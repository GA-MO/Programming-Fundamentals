# Open/Closed Principle (OCP) ใน C# .NET

## ความหมาย
**"เปิดให้ขยาย ปิดให้แก้ไข"** - โปรแกรมควรสามารถขยายฟีเจอร์ใหม่ได้โดยไม่ต้องแก้ไขโค้ดเดิม

## หลักการ
- **Open for Extension**: สามารถเพิ่มฟีเจอร์ใหม่ได้
- **Closed for Modification**: ไม่ต้องแก้ไขโค้ดเดิม
- ใช้ **interfaces** และ **polymorphism** เพื่อให้เกิดความยืดหยุ่น

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```csharp
// PaymentProcessor ที่ต้องแก้ไขทุกครั้งที่มี payment method ใหม่
using System;
using System.Threading.Tasks;

public class Payment
{
    public decimal Amount { get; set; }
    public string Method { get; set; }
}

public class PaymentProcessor
{
    public async Task ProcessPaymentAsync(Payment payment)
    {
        switch (payment.Method.ToLower())
        {
            case "credit_card":
                Console.WriteLine($"Processing credit card payment: ${payment.Amount}");
                await ValidateCreditCardAsync();
                await ChargeCreditCardAsync(payment.Amount);
                await SendReceiptAsync();
                break;

            case "paypal":
                Console.WriteLine($"Processing PayPal payment: ${payment.Amount}");
                await ValidatePayPalAsync();
                await ChargePayPalAsync(payment.Amount);
                await SendReceiptAsync();
                break;

            case "bitcoin":
                Console.WriteLine($"Processing Bitcoin payment: ${payment.Amount}");
                await ValidateBitcoinAsync();
                await ChargeBitcoinAsync(payment.Amount);
                await SendReceiptAsync();
                break;

            default:
                throw new ArgumentException("Unsupported payment method");
        }
    }

    private async Task ValidateCreditCardAsync()
    {
        Console.WriteLine("Validating credit card...");
        await Task.Delay(100);
    }

    private async Task ChargeCreditCardAsync(decimal amount)
    {
        Console.WriteLine($"Charging credit card: ${amount}");
        await Task.Delay(100);
    }

    private async Task ValidatePayPalAsync()
    {
        Console.WriteLine("Validating PayPal...");
        await Task.Delay(100);
    }

    private async Task ChargePayPalAsync(decimal amount)
    {
        Console.WriteLine($"Charging PayPal: ${amount}");
        await Task.Delay(100);
    }

    private async Task ValidateBitcoinAsync()
    {
        Console.WriteLine("Validating Bitcoin...");
        await Task.Delay(100);
    }

    private async Task ChargeBitcoinAsync(decimal amount)
    {
        Console.WriteLine($"Charging Bitcoin: ${amount}");
        await Task.Delay(100);
    }

    private async Task SendReceiptAsync()
    {
        Console.WriteLine("Sending receipt...");
        await Task.Delay(100);
    }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **ต้องแก้ไข PaymentProcessor** ทุกครั้งที่มี payment method ใหม่
2. **ยากต่อการทดสอบ**: ต้อง mock หลาย method
3. **เสี่ยงต่อการเกิด bug**: แก้ไขโค้ดเดิมอาจกระทบส่วนอื่น
4. **ขยายยาก**: ต้องเพิ่ม case ใหม่ใน switch statement

---

## ✅ ตัวอย่างที่ดี (Good Example)

```csharp
using System;
using System.Threading.Tasks;

public class Payment
{
    public decimal Amount { get; set; }
}

// Interface สำหรับ payment method ต่างๆ
public interface IPaymentMethod
{
    Task ProcessAsync(Payment payment);
    Task ValidateAsync();
    Task ChargeAsync(decimal amount);
}

// Credit Card Payment
public class CreditCardPayment : IPaymentMethod
{
    public async Task ProcessAsync(Payment payment)
    {
        Console.WriteLine($"Processing credit card payment: ${payment.Amount}");
        await ValidateAsync();
        await ChargeAsync(payment.Amount);
        await SendReceiptAsync();
    }

    public async Task ValidateAsync()
    {
        Console.WriteLine("Validating credit card...");
        await Task.Delay(100);
    }

    public async Task ChargeAsync(decimal amount)
    {
        Console.WriteLine($"Charging credit card: ${amount}");
        await Task.Delay(100);
    }

    private async Task SendReceiptAsync()
    {
        Console.WriteLine("Sending credit card receipt...");
        await Task.Delay(100);
    }
}

// PayPal Payment
public class PayPalPayment : IPaymentMethod
{
    public async Task ProcessAsync(Payment payment)
    {
        Console.WriteLine($"Processing PayPal payment: ${payment.Amount}");
        await ValidateAsync();
        await ChargeAsync(payment.Amount);
        await SendReceiptAsync();
    }

    public async Task ValidateAsync()
    {
        Console.WriteLine("Validating PayPal...");
        await Task.Delay(100);
    }

    public async Task ChargeAsync(decimal amount)
    {
        Console.WriteLine($"Charging PayPal: ${amount}");
        await Task.Delay(100);
    }

    private async Task SendReceiptAsync()
    {
        Console.WriteLine("Sending PayPal receipt...");
        await Task.Delay(100);
    }
}

// Bitcoin Payment
public class BitcoinPayment : IPaymentMethod
{
    public async Task ProcessAsync(Payment payment)
    {
        Console.WriteLine($"Processing Bitcoin payment: ${payment.Amount}");
        await ValidateAsync();
        await ChargeAsync(payment.Amount);
        await SendReceiptAsync();
    }

    public async Task ValidateAsync()
    {
        Console.WriteLine("Validating Bitcoin...");
        await Task.Delay(100);
    }

    public async Task ChargeAsync(decimal amount)
    {
        Console.WriteLine($"Charging Bitcoin: ${amount}");
        await Task.Delay(100);
    }

    private async Task SendReceiptAsync()
    {
        Console.WriteLine("Sending Bitcoin receipt...");
        await Task.Delay(100);
    }
}

// PaymentProcessor ที่ไม่ต้องแก้ไขเมื่อมี payment method ใหม่
public class PaymentProcessor
{
    public async Task ProcessPaymentAsync(Payment payment, IPaymentMethod method)
    {
        await method.ProcessAsync(payment);
    }
}
```

### ข้อดีของโค้ดที่ดี:
1. **ไม่ต้องแก้ไข PaymentProcessor**: เพิ่ม payment method ใหม่โดยไม่แตะต้องโค้ดเดิม
2. **ทดสอบง่าย**: สามารถ mock interface ได้ง่าย
3. **ขยายง่าย**: เพียง implement interface ใหม่
4. **แยกความรับผิดชอบ**: แต่ละ payment method จัดการตัวเอง

---

## การขยายฟีเจอร์ (Extension)

```csharp
// เพิ่ม payment method ใหม่โดยไม่แก้ไขโค้ดเดิม
public class ApplePayPayment : IPaymentMethod
{
    public async Task ProcessAsync(Payment payment)
    {
        Console.WriteLine($"Processing Apple Pay payment: ${payment.Amount}");
        await ValidateAsync();
        await ChargeAsync(payment.Amount);
        await SendReceiptAsync();
    }

    public async Task ValidateAsync()
    {
        Console.WriteLine("Validating Apple Pay...");
        await Task.Delay(100);
    }

    public async Task ChargeAsync(decimal amount)
    {
        Console.WriteLine($"Charging Apple Pay: ${amount}");
        await Task.Delay(100);
    }

    private async Task SendReceiptAsync()
    {
        Console.WriteLine("Sending Apple Pay receipt...");
        await Task.Delay(100);
    }
}
```

---

## ตัวอย่างการใช้งานจริง

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var processor = new PaymentProcessor();
        var payment = new Payment { Amount = 100 };

        // ใช้ payment method ต่างๆ
        var creditCard = new CreditCardPayment();
        await processor.ProcessPaymentAsync(payment, creditCard);

        var paypal = new PayPalPayment();
        await processor.ProcessPaymentAsync(payment, paypal);

        var bitcoin = new BitcoinPayment();
        await processor.ProcessPaymentAsync(payment, bitcoin);

        // เพิ่ม payment method ใหม่โดยไม่แก้ไขโค้ดเดิม
        var applePay = new ApplePayPayment();
        await processor.ProcessPaymentAsync(payment, applePay);
    }
}
```

---

## การทดสอบ (Testing)

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using Moq;
using System;
using System.Threading.Tasks;

[TestClass]
public class PaymentProcessorTests
{
    [TestMethod]
    public async Task ProcessPayment_WithValidMethod_ShouldSucceed()
    {
        // Arrange
        var processor = new PaymentProcessor();
        var payment = new Payment { Amount = 50 };
        var mockMethod = new Mock<IPaymentMethod>();

        // Act
        await processor.ProcessPaymentAsync(payment, mockMethod.Object);

        // Assert
        mockMethod.Verify(m => m.ProcessAsync(It.IsAny<Payment>()), Times.Once);
    }

    [TestMethod]
    public async Task CreditCardPayment_Process_ShouldSucceed()
    {
        // Arrange
        var creditCard = new CreditCardPayment();
        var payment = new Payment { Amount = 100 };

        // Act & Assert
        await creditCard.ProcessAsync(payment); // Should not throw
    }

    [TestMethod]
    public async Task PayPalPayment_Process_ShouldSucceed()
    {
        // Arrange
        var paypal = new PayPalPayment();
        var payment = new Payment { Amount = 75 };

        // Act & Assert
        await paypal.ProcessAsync(payment); // Should not throw
    }

    [TestMethod]
    public async Task BitcoinPayment_Process_ShouldSucceed()
    {
        // Arrange
        var bitcoin = new BitcoinPayment();
        var payment = new Payment { Amount = 200 };

        // Act & Assert
        await bitcoin.ProcessAsync(payment); // Should not throw
    }
}
```

---

## สรุป

**OCP ใน C# .NET ช่วยให้:**
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

- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก interface เล็กๆ ช่วยให้ OCP ยืดหยุ่นขึ้น 