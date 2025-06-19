# Open/Closed Principle (OCP) ใน TypeScript

## ความหมาย
**"เปิดให้ขยาย ปิดให้แก้ไข"** - โปรแกรมควรสามารถขยายฟีเจอร์ใหม่ได้โดยไม่ต้องแก้ไขโค้ดเดิม

## หลักการ
- **Open for Extension**: สามารถเพิ่มฟีเจอร์ใหม่ได้
- **Closed for Modification**: ไม่ต้องแก้ไขโค้ดเดิม
- ใช้ **interfaces** และ **polymorphism** เพื่อให้เกิดความยืดหยุ่น

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```typescript
// PaymentProcessor ที่ต้องแก้ไขทุกครั้งที่มี payment method ใหม่
interface Payment {
  amount: number;
  method: string;
}

class PaymentProcessor {
  processPayment(payment: Payment): Promise<void> {
    switch (payment.method) {
      case 'credit_card':
        console.log(`Processing credit card payment: $${payment.amount}`);
        this.validateCreditCard();
        this.chargeCreditCard(payment.amount);
        this.sendReceipt();
        break;
      
      case 'paypal':
        console.log(`Processing PayPal payment: $${payment.amount}`);
        this.validatePayPal();
        this.chargePayPal(payment.amount);
        this.sendReceipt();
        break;
      
      case 'bitcoin':
        console.log(`Processing Bitcoin payment: $${payment.amount}`);
        this.validateBitcoin();
        this.chargeBitcoin(payment.amount);
        this.sendReceipt();
        break;
      
      default:
        throw new Error('Unsupported payment method');
    }
    
    return Promise.resolve();
  }

  private validateCreditCard(): void {
    console.log('Validating credit card...');
  }

  private chargeCreditCard(amount: number): void {
    console.log(`Charging credit card: $${amount}`);
  }

  private validatePayPal(): void {
    console.log('Validating PayPal...');
  }

  private chargePayPal(amount: number): void {
    console.log(`Charging PayPal: $${amount}`);
  }

  private validateBitcoin(): void {
    console.log('Validating Bitcoin...');
  }

  private chargeBitcoin(amount: number): void {
    console.log(`Charging Bitcoin: $${amount}`);
  }

  private sendReceipt(): void {
    console.log('Sending receipt...');
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

```typescript
interface Payment {
  amount: number;
}

// Interface สำหรับ payment method ต่างๆ
interface PaymentMethod {
  process(payment: Payment): Promise<void>;
  validate(): Promise<void>;
  charge(amount: number): Promise<void>;
}

// Credit Card Payment
class CreditCardPayment implements PaymentMethod {
  async process(payment: Payment): Promise<void> {
    console.log(`Processing credit card payment: $${payment.amount}`);
    await this.validate();
    await this.charge(payment.amount);
    await this.sendReceipt();
  }

  async validate(): Promise<void> {
    console.log('Validating credit card...');
  }

  async charge(amount: number): Promise<void> {
    console.log(`Charging credit card: $${amount}`);
  }

  private async sendReceipt(): Promise<void> {
    console.log('Sending credit card receipt...');
  }
}

// PayPal Payment
class PayPalPayment implements PaymentMethod {
  async process(payment: Payment): Promise<void> {
    console.log(`Processing PayPal payment: $${payment.amount}`);
    await this.validate();
    await this.charge(payment.amount);
    await this.sendReceipt();
  }

  async validate(): Promise<void> {
    console.log('Validating PayPal...');
  }

  async charge(amount: number): Promise<void> {
    console.log(`Charging PayPal: $${amount}`);
  }

  private async sendReceipt(): Promise<void> {
    console.log('Sending PayPal receipt...');
  }
}

// Bitcoin Payment
class BitcoinPayment implements PaymentMethod {
  async process(payment: Payment): Promise<void> {
    console.log(`Processing Bitcoin payment: $${payment.amount}`);
    await this.validate();
    await this.charge(payment.amount);
    await this.sendReceipt();
  }

  async validate(): Promise<void> {
    console.log('Validating Bitcoin...');
  }

  async charge(amount: number): Promise<void> {
    console.log(`Charging Bitcoin: $${amount}`);
  }

  private async sendReceipt(): Promise<void> {
    console.log('Sending Bitcoin receipt...');
  }
}

// PaymentProcessor ที่ไม่ต้องแก้ไขเมื่อมี payment method ใหม่
class PaymentProcessor {
  processPayment(payment: Payment, method: PaymentMethod): Promise<void> {
    return method.process(payment);
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

```typescript
// เพิ่ม payment method ใหม่โดยไม่แก้ไขโค้ดเดิม
class ApplePayPayment implements PaymentMethod {
  async process(payment: Payment): Promise<void> {
    console.log(`Processing Apple Pay payment: $${payment.amount}`);
    await this.validate();
    await this.charge(payment.amount);
    await this.sendReceipt();
  }

  async validate(): Promise<void> {
    console.log('Validating Apple Pay...');
  }

  async charge(amount: number): Promise<void> {
    console.log(`Charging Apple Pay: $${amount}`);
  }

  private async sendReceipt(): Promise<void> {
    console.log('Sending Apple Pay receipt...');
  }
}
```

---

## ตัวอย่างการใช้งานจริง

```typescript
async function main(): Promise<void> {
  const processor = new PaymentProcessor();
  const payment: Payment = { amount: 100 };

  // ใช้ payment method ต่างๆ
  const creditCard = new CreditCardPayment();
  await processor.processPayment(payment, creditCard);

  const paypal = new PayPalPayment();
  await processor.processPayment(payment, paypal);

  const bitcoin = new BitcoinPayment();
  await processor.processPayment(payment, bitcoin);

  // เพิ่ม payment method ใหม่โดยไม่แก้ไขโค้ดเดิม
  const applePay = new ApplePayPayment();
  await processor.processPayment(payment, applePay);
}

main().catch(console.error);
```

---

## การทดสอบ (Testing)

```typescript
// Mock PaymentMethod สำหรับการทดสอบ
class MockPaymentMethod implements PaymentMethod {
  private shouldFail: boolean;

  constructor(shouldFail: boolean = false) {
    this.shouldFail = shouldFail;
  }

  async process(payment: Payment): Promise<void> {
    if (this.shouldFail) {
      throw new Error('Mock payment failed');
    }
    console.log(`Mock payment processed: $${payment.amount}`);
  }

  async validate(): Promise<void> {
    if (this.shouldFail) {
      throw new Error('Mock validation failed');
    }
  }

  async charge(amount: number): Promise<void> {
    if (this.shouldFail) {
      throw new Error('Mock charge failed');
    }
  }
}

// Tests
describe('PaymentProcessor', () => {
  test('should process payment successfully', async () => {
    const processor = new PaymentProcessor();
    const payment: Payment = { amount: 50 };
    const mockMethod = new MockPaymentMethod(false);

    await expect(processor.processPayment(payment, mockMethod)).resolves.not.toThrow();
  });

  test('should handle payment failure', async () => {
    const processor = new PaymentProcessor();
    const payment: Payment = { amount: 50 };
    const mockMethod = new MockPaymentMethod(true);

    await expect(processor.processPayment(payment, mockMethod)).rejects.toThrow('Mock payment failed');
  });
});

describe('CreditCardPayment', () => {
  test('should process credit card payment', async () => {
    const creditCard = new CreditCardPayment();
    const payment: Payment = { amount: 100 };

    await expect(creditCard.process(payment)).resolves.not.toThrow();
  });
});

describe('PayPalPayment', () => {
  test('should process PayPal payment', async () => {
    const paypal = new PayPalPayment();
    const payment: Payment = { amount: 75 };

    await expect(paypal.process(payment)).resolves.not.toThrow();
  });
});
```

---

## สรุป

**OCP ใน TypeScript ช่วยให้:**
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