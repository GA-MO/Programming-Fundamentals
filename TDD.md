# Test-Driven Development (TDD)

## TDD คืออะไร?

Test-Driven Development (TDD) เป็นแนวทางการพัฒนาซอฟต์แวร์ที่เขียน Test ก่อนเขียน Code โดยมีหลักการง่ายๆ คือ **"เขียน Test ที่ Fail ก่อน แล้วค่อยเขียน Code ให้ Test ผ่าน"**

## หลักการ Red-Green-Refactor

TDD ทำงานตามรอบ 3 ขั้นตอน:

1. **🔴 Red**: เขียน Test ที่ Fail (ยังไม่มี Code ที่ใช้งานได้)
2. **🟢 Green**: เขียน Code เพื่อให้ Test ผ่าน (ใช้วิธีง่ายที่สุด)
3. **🔵 Refactor**: ปรับปรุง Code ให้สะอาดขึ้น โดยไม่ทำให้ Test Fail

## ประโยชน์ของ TDD

- **คุณภาพสูง**: Code มี Test Coverage สูง ลด Bug
- **Design ที่ดี**: บังคับให้คิด Interface ก่อน Implementation
- **มั่นใจในการแก้ไข**: มี Test คุ้มครอง เปลี่ยน Code ได้อย่างปลอดภัย
- **Documentation**: Test เป็น Documentation ที่มีชีวิต

## ตัวอย่างที่ดี ✅

### การทำ TDD อย่างถูกต้อง

```go
// 1. Red: เขียน Test ที่ Fail ก่อน
// password_validator_test.go
package main

import "testing"

func TestPasswordValidator_ShouldReturnTrueForValidPassword(t *testing.T) {
    validator := PasswordValidator{}
    result := validator.IsValid("mypassword123")
    
    if !result {
        t.Error("IsValid('mypassword123') = false; want true")
    }
}

// 2. Green: เขียน Code ให้ Test ผ่าน (วิธีง่ายที่สุด)
// password_validator.go
package main

type PasswordValidator struct{}

func (p PasswordValidator) IsValid(password string) bool {
    return true  // Simple implementation to make test pass
}

// 3. Refactor: เพิ่มเงื่อนไขจริง
func (p PasswordValidator) IsValid(password string) bool {
    if len(password) < 8 {
        return false
    }
    if len(password) > 50 {
        return false
    }
    return true
}
```

### คุณลักษณะของ Test ที่ดี

```go
// ชื่อ Test บอกความต้องการชัดเจน
func TestUserService_ShouldReturnUserWhenValidEmailProvided(t *testing.T) {
    // Arrange
    userService := NewUserService()
    email := "john@example.com"
    
    // Act
    user, err := userService.FindByEmail(email)
    
    // Assert
    if err != nil {
        t.Errorf("FindByEmail() error = %v", err)
        return
    }
    if user.Email != email {
        t.Errorf("FindByEmail() user.Email = %s; want %s", user.Email, email)
    }
    if user == nil {
        t.Error("FindByEmail() returned nil user")
    }
}
```

## ตัวอย่างที่ไม่ดี ❌

### Anti-patterns ที่ควรหลีกเลี่ยง

#### 1. เขียน Code ก่อน แล้วค่อยเขียน Test

```go
// ❌ ผิด: เขียน Code ก่อน
type ProductService struct{}

func (p ProductService) CalculatePrice(basePrice float64, discount float64) float64 {
    return basePrice - (basePrice * discount / 100)
}

func (p ProductService) IsPremiumCustomer(customerID string) bool {
    // ตรวจสอบจาก Database...
    return customerID == "PREMIUM"
}

// แล้วค่อยเขียน Test ทีหลัง (ไม่ใช่ TDD)
func TestProductServiceWorks(t *testing.T) {
    service := ProductService{}
    result := service.CalculatePrice(100.0, 10.0)
    if result != 90.0 {
        t.Errorf("CalculatePrice(100, 10) = %f; want 90.0", result)
    }
}
```

#### 2. Test ที่ซับซ้อนเกินไป

```go
// ❌ ผิด: Test ซับซ้อน ทดสอบหลายอย่างพร้อมกัน
func TestOrderProcessing(t *testing.T) {
    order := Order{
        Items: []Item{{Name: "Laptop", Price: 1000}},
        CustomerID: "CUST123",
        DiscountCode: "SAVE10",
    }
    
    // ทำหลายอย่างใน Test เดียว
    validator := OrderValidator{}
    isValid := validator.ValidateOrder(order)
    
    calculator := PriceCalculator{}
    finalPrice := calculator.CalculateTotal(order)
    
    notification := NotificationService{}
    notification.SendConfirmation(order)
    
    // Assert หลายเรื่องพร้อมกัน
    if !isValid {
        t.Error("Order should be valid")
    }
    if finalPrice != 900.0 {
        t.Errorf("Final price = %f; want 900.0", finalPrice)
    }
    // การ test notification ขึ้นอยู่กับ external service
}
```

#### 3. Test ขึ้นอยู่กับ External Dependencies

```go
// ❌ ผิด: Test ขึ้นอยู่กับ Database จริง
func TestInventoryService_ShouldUpdateStockAfterPurchase(t *testing.T) {
    product := Product{ID: "LAPTOP001", Stock: 10}
    
    // ขึ้นอยู่กับ DB จริง - ไม่ควรทำ
    err := realDatabase.UpdateStock(product.ID, 5)
    
    if err != nil {
        t.Errorf("UpdateStock() error = %v", err)
    }
    
    // ต้องไปเช็คใน DB จริงอีก
    updatedProduct, _ := realDatabase.GetProduct(product.ID)
    if updatedProduct.Stock != 5 {
        t.Errorf("Stock = %d; want 5", updatedProduct.Stock)
    }
}
```

## เคล็ดลับการใช้ TDD

### 1. เริ่มจาก Test ที่ง่ายที่สุด

```go
// ✅ ดี: เริ่มจาก Happy Path ง่ายๆ
func TestEmailValidator_ShouldReturnFalseForEmptyEmail(t *testing.T) {
    validator := EmailValidator{}
    
    result := validator.IsValid("")
    
    if result {
        t.Error("IsValid('') = true; want false")
    }
}
```

### 2. ใช้ Mock/Stub สำหรับ Dependencies

```go
// ✅ ดี: ใช้ Mock แทน External Dependencies
type MockPaymentGateway struct {
    ProcessCalled bool
    ProcessParams PaymentRequest
    ProcessResult PaymentResponse
}

func (m *MockPaymentGateway) ProcessPayment(req PaymentRequest) (PaymentResponse, error) {
    m.ProcessCalled = true
    m.ProcessParams = req
    return m.ProcessResult, nil
}

func TestOrderService_ShouldProcessPaymentWhenOrderCreated(t *testing.T) {
    mockPayment := &MockPaymentGateway{
        ProcessResult: PaymentResponse{TransactionID: "TXN123", Status: "SUCCESS"},
    }
    orderService := NewOrderService(mockPayment)
    
    order := Order{TotalAmount: 100.0, CustomerEmail: "john@example.com"}
    result, err := orderService.CreateOrder(order)
    
    if err != nil {
        t.Errorf("CreateOrder() error = %v", err)
    }
    if !mockPayment.ProcessCalled {
        t.Error("Expected ProcessPayment() to be called")
    }
    if mockPayment.ProcessParams.Amount != 100.0 {
        t.Errorf("ProcessPayment() amount = %f; want 100.0", mockPayment.ProcessParams.Amount)
    }
    if result.Status != "COMPLETED" {
        t.Errorf("CreateOrder() status = %s; want COMPLETED", result.Status)
    }
}
```

### 3. Test One Thing at a Time

```go
// ✅ ดี: แยก Test แต่ละ Case
func TestLoginValidator_ShouldReturnTrueForValidCredentials(t *testing.T) {
    validator := LoginValidator{}
    
    result := validator.IsValidUsername("john_doe")
    
    if !result {
        t.Error("IsValidUsername('john_doe') = false; want true")
    }
}

func TestLoginValidator_ShouldReturnFalseForShortUsername(t *testing.T) {
    validator := LoginValidator{}
    
    result := validator.IsValidUsername("ab")
    
    if result {
        t.Error("IsValidUsername('ab') = true; want false")
    }
}

func TestLoginValidator_ShouldReturnFalseForUsernameWithSpecialChars(t *testing.T) {
    validator := LoginValidator{}
    
    result := validator.IsValidUsername("user@name")
    
    if result {
        t.Error("IsValidUsername('user@name') = true; want false")
    }
}
```

## สรุป

TDD ไม่ใช่แค่การเขียน Test แต่เป็นการเปลี่ยนวิธีคิดในการพัฒนา Software ให้มีคุณภาพสูงขึ้น โดย:

- **เขียน Test ก่อนเสมอ** (Red-Green-Refactor)
- **Test ต้องง่าย ชัดเจน และเป็นอิสระ**
- **เขียน Code เพื่อให้ Test ผ่าน แล้วค่อย Refactor**
- **หลีกเลี่ยง Complex Test และ External Dependencies**

การฝึกฝน TDD อย่างสม่ำเสมอจะทำให้เราเขียน Code ที่มีคุณภาพสูง บำรุงรักษาง่าย และมั่นใจในการแก้ไขได้มากขึ้น
