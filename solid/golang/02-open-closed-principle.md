# Open/Closed Principle (OCP)

## ความหมาย
**"เปิดให้ขยาย ปิดให้แก้ไข"** - ซอฟต์แวร์ควรเปิดให้ขยายฟีเจอร์ใหม่ได้ แต่ปิดให้แก้ไขโค้ดเดิม

## หลักการ
- **Open for Extension**: สามารถเพิ่มฟีเจอร์ใหม่ได้โดยไม่แก้ไขโค้ดเดิม
- **Closed for Modification**: โค้ดเดิมไม่ควรถูกแก้ไขเมื่อเพิ่มฟีเจอร์ใหม่
- ใช้ **polymorphism** และ **abstraction** เพื่อให้ขยายได้

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```go
// PaymentProcessor ที่ต้องแก้ไขทุกครั้งที่มี payment method ใหม่
type PaymentProcessor struct{}

func (p *PaymentProcessor) ProcessPayment(paymentType string, amount float64) error {
    switch paymentType {
    case "credit_card":
        return p.processCreditCard(amount)
    case "debit_card":
        return p.processDebitCard(amount)
    case "bank_transfer":
        return p.processBankTransfer(amount)
    case "crypto": // เพิ่มใหม่ - ต้องแก้ไขโค้ดเดิม!
        return p.processCrypto(amount)
    default:
        return errors.New("unsupported payment method")
    }
}

func (p *PaymentProcessor) processCreditCard(amount float64) error {
    fmt.Printf("Processing credit card payment: $%.2f\n", amount)
    return nil
}

func (p *PaymentProcessor) processDebitCard(amount float64) error {
    fmt.Printf("Processing debit card payment: $%.2f\n", amount)
    return nil
}

func (p *PaymentProcessor) processBankTransfer(amount float64) error {
    fmt.Printf("Processing bank transfer: $%.2f\n", amount)
    return nil
}

func (p *PaymentProcessor) processCrypto(amount float64) error {
    fmt.Printf("Processing crypto payment: $%.2f\n", amount)
    return nil
}
```

### ปัญหาของโค้ดข้างต้น:
1. **ต้องแก้ไขโค้ดเดิม**: ทุกครั้งที่มี payment method ใหม่
2. **เสี่ยงต่อการทำลายโค้ดเดิม**: อาจทำให้ payment method เดิมเสียหาย
3. **ยากต่อการทดสอบ**: ต้องทดสอบ switch case ทั้งหมดใหม่
4. **ละเมิด SRP**: PaymentProcessor รู้จัก payment method ทุกประเภท

---

## ✅ ตัวอย่างที่ดี (Good Example)

```go
// ใช้ interface และ polymorphism

// PaymentMethod interface - abstraction
type PaymentMethod interface {
    Process(amount float64) error
    GetName() string
}

// CreditCard implementation
type CreditCard struct {
    cardNumber string
    expiryDate string
    cvv        string
}

func (c *CreditCard) Process(amount float64) error {
    fmt.Printf("Processing credit card payment: $%.2f\n", amount)
    return nil
}

func (c *CreditCard) GetName() string {
    return "credit_card"
}

// DebitCard implementation
type DebitCard struct {
    cardNumber string
    pin        string
}

func (d *DebitCard) Process(amount float64) error {
    fmt.Printf("Processing debit card payment: $%.2f\n", amount)
    return nil
}

func (d *DebitCard) GetName() string {
    return "debit_card"
}

// BankTransfer implementation
type BankTransfer struct {
    accountNumber string
    routingNumber string
}

func (b *BankTransfer) Process(amount float64) error {
    fmt.Printf("Processing bank transfer: $%.2f\n", amount)
    return nil
}

func (b *BankTransfer) GetName() string {
    return "bank_transfer"
}

// Crypto implementation - เพิ่มใหม่โดยไม่แก้ไขโค้ดเดิม
type Crypto struct {
    walletAddress string
    coinType      string
}

func (c *Crypto) Process(amount float64) error {
    fmt.Printf("Processing crypto payment: $%.2f\n", amount)
    return nil
}

func (c *Crypto) GetName() string {
    return "crypto"
}

// PaymentProcessor ที่ไม่ต้องแก้ไข
type PaymentProcessor struct{}

func (p *PaymentProcessor) ProcessPayment(paymentMethod PaymentMethod, amount float64) error {
    return paymentMethod.Process(amount)
}
```

### ข้อดีของโค้ดที่ดี:
1. **ไม่ต้องแก้ไขโค้ดเดิม**: เพิ่ม payment method ใหม่โดยไม่แตะต้อง PaymentProcessor
2. **ปลอดภัย**: โค้ดเดิมไม่เสี่ยงต่อการเสียหาย
3. **ทดสอบง่าย**: ทดสอบแต่ละ payment method แยกกัน
4. **ขยายง่าย**: เพิ่ม payment method ใหม่ได้ไม่จำกัด

---

## ตัวอย่างการใช้งานจริง

```go
func main() {
    processor := &PaymentProcessor{}
    
    // ใช้งาน payment methods ต่างๆ
    creditCard := &CreditCard{
        cardNumber: "1234-5678-9012-3456",
        expiryDate: "12/25",
        cvv:        "123",
    }
    
    debitCard := &DebitCard{
        cardNumber: "9876-5432-1098-7654",
        pin:        "1234",
    }
    
    crypto := &Crypto{
        walletAddress: "0x1234567890abcdef",
        coinType:      "ETH",
    }
    
    // Process payments
    amount := 100.50
    
    processor.ProcessPayment(creditCard, amount)
    processor.ProcessPayment(debitCard, amount)
    processor.ProcessPayment(crypto, amount) // เพิ่มใหม่โดยไม่แก้ไขโค้ดเดิม
}
```

---

## การเพิ่ม Payment Method ใหม่

```go
// เพิ่ม PayPal โดยไม่แก้ไขโค้ดเดิม
type PayPal struct {
    email    string
    password string
}

func (p *PayPal) Process(amount float64) error {
    fmt.Printf("Processing PayPal payment: $%.2f\n", amount)
    return nil
}

func (p *PayPal) GetName() string {
    return "paypal"
}

// ใช้งานได้ทันทีโดยไม่แก้ไข PaymentProcessor
func main() {
    processor := &PaymentProcessor{}
    paypal := &PayPal{
        email:    "user@example.com",
        password: "password123",
    }
    
    processor.ProcessPayment(paypal, 50.00) // ใช้งานได้เลย!
}
```

---

## การใช้ Factory Pattern

```go
// PaymentMethodFactory - สร้าง payment method ตาม type
type PaymentMethodFactory struct{}

func (f *PaymentMethodFactory) CreatePaymentMethod(paymentType string, config map[string]string) (PaymentMethod, error) {
    switch paymentType {
    case "credit_card":
        return &CreditCard{
            cardNumber: config["card_number"],
            expiryDate: config["expiry_date"],
            cvv:        config["cvv"],
        }, nil
    case "debit_card":
        return &DebitCard{
            cardNumber: config["card_number"],
            pin:        config["pin"],
        }, nil
    case "crypto":
        return &Crypto{
            walletAddress: config["wallet_address"],
            coinType:      config["coin_type"],
        }, nil
    case "paypal":
        return &PayPal{
            email:    config["email"],
            password: config["password"],
        }, nil
    default:
        return nil, fmt.Errorf("unsupported payment method: %s", paymentType)
    }
}

// ใช้งาน
func main() {
    factory := &PaymentMethodFactory{}
    processor := &PaymentProcessor{}
    
    // สร้าง payment method จาก config
    config := map[string]string{
        "card_number": "1234-5678-9012-3456",
        "expiry_date": "12/25",
        "cvv":        "123",
    }
    
    paymentMethod, err := factory.CreatePaymentMethod("credit_card", config)
    if err != nil {
        log.Fatal(err)
    }
    
    processor.ProcessPayment(paymentMethod, 100.00)
}
```

---

## การทดสอบ (Testing)

```go
func TestPaymentProcessor_ProcessPayment(t *testing.T) {
    processor := &PaymentProcessor{}
    
    // Mock payment method
    mockPayment := &MockPaymentMethod{}
    mockPayment.On("Process", 100.00).Return(nil)
    
    // Test
    err := processor.ProcessPayment(mockPayment, 100.00)
    
    // Assert
    assert.NoError(t, err)
    mockPayment.AssertExpectations(t)
}

func TestCreditCard_Process(t *testing.T) {
    creditCard := &CreditCard{
        cardNumber: "1234-5678-9012-3456",
        expiryDate: "12/25",
        cvv:        "123",
    }
    
    err := creditCard.Process(100.00)
    
    assert.NoError(t, err)
    assert.Equal(t, "credit_card", creditCard.GetName())
}
```

---

## สรุป

**OCP ช่วยให้:**
- เพิ่มฟีเจอร์ใหม่ได้โดยไม่แก้ไขโค้ดเดิม
- ลดความเสี่ยงในการทำลายโค้ดเดิม
- เพิ่มความยืดหยุ่นในการขยายระบบ
- ง่ายต่อการทดสอบและดูแลรักษา

**เมื่อไหร่ควรใช้:**
- เมื่อคาดว่าจะมีฟีเจอร์ใหม่เพิ่มเข้ามา
- เมื่อต้องการลดความเสี่ยงในการแก้ไขโค้ดเดิม
- เมื่อต้องการให้ระบบขยายได้ง่าย

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Liskov Substitution Principle](./03-liskov-substitution-principle.md)** - การใช้ polymorphism ต้องทำตาม LSP
- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - แต่ละ payment method มีหน้าที่เดียว 