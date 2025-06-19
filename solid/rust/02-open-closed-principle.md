# Open/Closed Principle (OCP) ใน Rust

## ความหมาย
**"เปิดให้ขยาย ปิดให้แก้ไข"** - โปรแกรมควรสามารถขยายฟีเจอร์ใหม่ได้โดยไม่ต้องแก้ไขโค้ดเดิม

## หลักการ
- **Open for Extension**: สามารถเพิ่มฟีเจอร์ใหม่ได้
- **Closed for Modification**: ไม่ต้องแก้ไขโค้ดเดิม
- ใช้ **traits** และ **polymorphism** เพื่อให้เกิดความยืดหยุ่น

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```rust
// PaymentProcessor ที่ต้องแก้ไขทุกครั้งที่มี payment method ใหม่
use std::error::Error;

#[derive(Debug)]
struct Payment {
    amount: f64,
    method: String,
}

struct PaymentProcessor;

impl PaymentProcessor {
    fn process_payment(&self, payment: &Payment) -> Result<(), Box<dyn Error>> {
        match payment.method.as_str() {
            "credit_card" => {
                println!("Processing credit card payment: ${}", payment.amount);
                self.validate_credit_card()?;
                self.charge_credit_card(payment.amount)?;
                self.send_receipt()?;
            }
            "paypal" => {
                println!("Processing PayPal payment: ${}", payment.amount);
                self.validate_paypal()?;
                self.charge_paypal(payment.amount)?;
                self.send_receipt()?;
            }
            "bitcoin" => {
                println!("Processing Bitcoin payment: ${}", payment.amount);
                self.validate_bitcoin()?;
                self.charge_bitcoin(payment.amount)?;
                self.send_receipt()?;
            }
            _ => return Err("Unsupported payment method".into()),
        }
        Ok(())
    }

    fn validate_credit_card(&self) -> Result<(), Box<dyn Error>> {
        println!("Validating credit card...");
        Ok(())
    }

    fn charge_credit_card(&self, amount: f64) -> Result<(), Box<dyn Error>> {
        println!("Charging credit card: ${}", amount);
        Ok(())
    }

    fn validate_paypal(&self) -> Result<(), Box<dyn Error>> {
        println!("Validating PayPal...");
        Ok(())
    }

    fn charge_paypal(&self, amount: f64) -> Result<(), Box<dyn Error>> {
        println!("Charging PayPal: ${}", amount);
        Ok(())
    }

    fn validate_bitcoin(&self) -> Result<(), Box<dyn Error>> {
        println!("Validating Bitcoin...");
        Ok(())
    }

    fn charge_bitcoin(&self, amount: f64) -> Result<(), Box<dyn Error>> {
        println!("Charging Bitcoin: ${}", amount);
        Ok(())
    }

    fn send_receipt(&self) -> Result<(), Box<dyn Error>> {
        println!("Sending receipt...");
        Ok(())
    }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **ต้องแก้ไข PaymentProcessor** ทุกครั้งที่มี payment method ใหม่
2. **ยากต่อการทดสอบ**: ต้อง mock หลาย method
3. **เสี่ยงต่อการเกิด bug**: แก้ไขโค้ดเดิมอาจกระทบส่วนอื่น
4. **ขยายยาก**: ต้องเพิ่ม case ใหม่ใน match statement

---

## ✅ ตัวอย่างที่ดี (Good Example)

```rust
use std::error::Error;

#[derive(Debug)]
struct Payment {
    amount: f64,
}

// Trait สำหรับ payment method ต่างๆ
trait PaymentMethod {
    fn process(&self, payment: &Payment) -> Result<(), Box<dyn Error>>;
    fn validate(&self) -> Result<(), Box<dyn Error>>;
    fn charge(&self, amount: f64) -> Result<(), Box<dyn Error>>;
}

// Credit Card Payment
struct CreditCardPayment;

impl PaymentMethod for CreditCardPayment {
    fn process(&self, payment: &Payment) -> Result<(), Box<dyn Error>> {
        println!("Processing credit card payment: ${}", payment.amount);
        self.validate()?;
        self.charge(payment.amount)?;
        self.send_receipt()?;
        Ok(())
    }

    fn validate(&self) -> Result<(), Box<dyn Error>> {
        println!("Validating credit card...");
        Ok(())
    }

    fn charge(&self, amount: f64) -> Result<(), Box<dyn Error>> {
        println!("Charging credit card: ${}", amount);
        Ok(())
    }
}

impl CreditCardPayment {
    fn send_receipt(&self) -> Result<(), Box<dyn Error>> {
        println!("Sending credit card receipt...");
        Ok(())
    }
}

// PayPal Payment
struct PayPalPayment;

impl PaymentMethod for PayPalPayment {
    fn process(&self, payment: &Payment) -> Result<(), Box<dyn Error>> {
        println!("Processing PayPal payment: ${}", payment.amount);
        self.validate()?;
        self.charge(payment.amount)?;
        self.send_receipt()?;
        Ok(())
    }

    fn validate(&self) -> Result<(), Box<dyn Error>> {
        println!("Validating PayPal...");
        Ok(())
    }

    fn charge(&self, amount: f64) -> Result<(), Box<dyn Error>> {
        println!("Charging PayPal: ${}", amount);
        Ok(())
    }
}

impl PayPalPayment {
    fn send_receipt(&self) -> Result<(), Box<dyn Error>> {
        println!("Sending PayPal receipt...");
        Ok(())
    }
}

// Bitcoin Payment
struct BitcoinPayment;

impl PaymentMethod for BitcoinPayment {
    fn process(&self, payment: &Payment) -> Result<(), Box<dyn Error>> {
        println!("Processing Bitcoin payment: ${}", payment.amount);
        self.validate()?;
        self.charge(payment.amount)?;
        self.send_receipt()?;
        Ok(())
    }

    fn validate(&self) -> Result<(), Box<dyn Error>> {
        println!("Validating Bitcoin...");
        Ok(())
    }

    fn charge(&self, amount: f64) -> Result<(), Box<dyn Error>> {
        println!("Charging Bitcoin: ${}", amount);
        Ok(())
    }
}

impl BitcoinPayment {
    fn send_receipt(&self) -> Result<(), Box<dyn Error>> {
        println!("Sending Bitcoin receipt...");
        Ok(())
    }
}

// PaymentProcessor ที่ไม่ต้องแก้ไขเมื่อมี payment method ใหม่
struct PaymentProcessor;

impl PaymentProcessor {
    fn process_payment(&self, payment: &Payment, method: &dyn PaymentMethod) -> Result<(), Box<dyn Error>> {
        method.process(payment)
    }
}
```

### ข้อดีของโค้ดที่ดี:
1. **ไม่ต้องแก้ไข PaymentProcessor**: เพิ่ม payment method ใหม่โดยไม่แตะต้องโค้ดเดิม
2. **ทดสอบง่าย**: สามารถ mock trait ได้ง่าย
3. **ขยายง่าย**: เพียง implement trait ใหม่
4. **แยกความรับผิดชอบ**: แต่ละ payment method จัดการตัวเอง

---

## การขยายฟีเจอร์ (Extension)

```rust
// เพิ่ม payment method ใหม่โดยไม่แก้ไขโค้ดเดิม
struct ApplePayPayment;

impl PaymentMethod for ApplePayPayment {
    fn process(&self, payment: &Payment) -> Result<(), Box<dyn Error>> {
        println!("Processing Apple Pay payment: ${}", payment.amount);
        self.validate()?;
        self.charge(payment.amount)?;
        self.send_receipt()?;
        Ok(())
    }

    fn validate(&self) -> Result<(), Box<dyn Error>> {
        println!("Validating Apple Pay...");
        Ok(())
    }

    fn charge(&self, amount: f64) -> Result<(), Box<dyn Error>> {
        println!("Charging Apple Pay: ${}", amount);
        Ok(())
    }
}

impl ApplePayPayment {
    fn send_receipt(&self) -> Result<(), Box<dyn Error>> {
        println!("Sending Apple Pay receipt...");
        Ok(())
    }
}
```

---

## ตัวอย่างการใช้งานจริง

```rust
fn main() -> Result<(), Box<dyn Error>> {
    let processor = PaymentProcessor;
    let payment = Payment { amount: 100.0 };

    // ใช้ payment method ต่างๆ
    let credit_card = CreditCardPayment;
    processor.process_payment(&payment, &credit_card)?;

    let paypal = PayPalPayment;
    processor.process_payment(&payment, &paypal)?;

    let bitcoin = BitcoinPayment;
    processor.process_payment(&payment, &bitcoin)?;

    // เพิ่ม payment method ใหม่โดยไม่แก้ไขโค้ดเดิม
    let apple_pay = ApplePayPayment;
    processor.process_payment(&payment, &apple_pay)?;

    Ok(())
}
```

---

## การทดสอบ (Testing)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Mock PaymentMethod สำหรับการทดสอบ
    struct MockPaymentMethod {
        should_fail: bool,
    }

    impl PaymentMethod for MockPaymentMethod {
        fn process(&self, payment: &Payment) -> Result<(), Box<dyn Error>> {
            if self.should_fail {
                return Err("Mock payment failed".into());
            }
            println!("Mock payment processed: ${}", payment.amount);
            Ok(())
        }

        fn validate(&self) -> Result<(), Box<dyn Error>> {
            if self.should_fail {
                return Err("Mock validation failed".into());
            }
            Ok(())
        }

        fn charge(&self, amount: f64) -> Result<(), Box<dyn Error>> {
            if self.should_fail {
                return Err("Mock charge failed".into());
            }
            Ok(())
        }
    }

    #[test]
    fn test_payment_processor_success() {
        let processor = PaymentProcessor;
        let payment = Payment { amount: 50.0 };
        let mock_method = MockPaymentMethod { should_fail: false };

        let result = processor.process_payment(&payment, &mock_method);
        assert!(result.is_ok());
    }

    #[test]
    fn test_payment_processor_failure() {
        let processor = PaymentProcessor;
        let payment = Payment { amount: 50.0 };
        let mock_method = MockPaymentMethod { should_fail: true };

        let result = processor.process_payment(&payment, &mock_method);
        assert!(result.is_err());
    }

    #[test]
    fn test_credit_card_payment() {
        let credit_card = CreditCardPayment;
        let payment = Payment { amount: 100.0 };

        let result = credit_card.process(&payment);
        assert!(result.is_ok());
    }

    #[test]
    fn test_paypal_payment() {
        let paypal = PayPalPayment;
        let payment = Payment { amount: 75.0 };

        let result = paypal.process(&payment);
        assert!(result.is_ok());
    }
}
```

---

## สรุป

**OCP ใน Rust ช่วยให้:**
- เพิ่มฟีเจอร์ใหม่โดยไม่แก้ไขโค้ดเดิม
- ลดความเสี่ยงในการเกิด bug
- เพิ่มความยืดหยุ่นของระบบ
- ง่ายต่อการทดสอบและดูแลรักษา

**เมื่อไหร่ควรใช้:**
- เมื่อต้องการเพิ่มฟีเจอร์ใหม่บ่อยๆ
- เมื่อไม่ต้องการแก้ไขโค้ดเดิม
- เมื่อต้องการความยืดหยุ่นสูง

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ trait ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก trait เล็กๆ ช่วยให้ OCP ยืดหยุ่นขึ้น 