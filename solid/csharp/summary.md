# สรุป SOLID Design Principles ใน C# .NET

## ภาพรวม

SOLID Design Principles เป็นหลักการออกแบบซอฟต์แวร์ที่ช่วยให้โค้ดมีคุณภาพสูง ง่ายต่อการดูแลรักษา และขยายได้ เอกสารนี้สรุปหลักการทั้ง 5 ข้อในบริบทของ C# .NET

---

## หลักการทั้ง 5 ข้อ

### 1. Single Responsibility Principle (SRP)
**"หนึ่ง class/interface ควรมีหน้าที่เดียว"**

**ใน C# .NET:**
- ใช้ **static class** สำหรับ utility functions
- แยก **interface** ตามหน้าที่
- ใช้ **Dependency Injection** เพื่อลดการพึ่งพา

**ตัวอย่าง:**
```csharp
// แยก UserValidator, PasswordHasher, UserRepository
public class UserService
{
    private readonly IUserRepository _repository;
    private readonly IEmailService _emailService;
    // แต่ละ dependency มีหน้าที่เดียว
}
```

### 2. Open/Closed Principle (OCP)
**"เปิดให้ขยาย ปิดให้แก้ไข"**

**ใน C# .NET:**
- ใช้ **interface** และ **polymorphism**
- ใช้ **extension methods** เพื่อขยาย functionality
- ใช้ **strategy pattern** สำหรับ algorithm ต่างๆ

**ตัวอย่าง:**
```csharp
public interface IPaymentMethod
{
    Task ProcessAsync(Payment payment);
}

// เพิ่ม payment method ใหม่โดยไม่แก้ไข PaymentProcessor
public class ApplePayPayment : IPaymentMethod { }
```

### 3. Liskov Substitution Principle (LSP)
**"Subtype ต้องสามารถแทนที่ base type ได้"**

**ใน C# .NET:**
- ใช้ **interface** แทน inheritance
- หลีกเลี่ยง **virtual methods** ที่เปลี่ยนพฤติกรรม
- ใช้ **contract** ที่ชัดเจน

**ตัวอย่าง:**
```csharp
public interface IShape
{
    int GetArea();
}

// Rectangle และ Square สามารถแทนที่ IShape ได้
public class Rectangle : IShape { }
public class Square : IShape { }
```

### 4. Interface Segregation Principle (ISP)
**"Client ไม่ควรถูกบังคับให้ขึ้นต่อ interface ที่ไม่ใช้"**

**ใน C# .NET:**
- แยก **interface** ให้เล็กลง
- ใช้ **multiple interface inheritance**
- หลีกเลี่ยง **fat interface**

**ตัวอย่าง:**
```csharp
public interface IWorkable { void Work(); }
public interface IEatable { void Eat(); }
public interface ISleepable { void Sleep(); }

// Employee implement หลาย interface
public class Employee : IWorkable, IEatable, ISleepable { }
```

### 5. Dependency Inversion Principle (DIP)
**"ขึ้นต่อ abstraction ไม่ใช่ concrete"**

**ใน C# .NET:**
- ใช้ **interface** และ **abstract class**
- ใช้ **Dependency Injection Container**
- ใช้ **constructor injection**

**ตัวอย่าง:**
```csharp
public class UserService
{
    private readonly IUserRepository _repository;
    
    public UserService(IUserRepository repository) // Dependency Injection
    {
        _repository = repository;
    }
}
```

---

## คุณสมบัติพิเศษของ C# .NET ที่สนับสนุน SOLID

### 1. Interfaces
- **Multiple interface inheritance**
- **Explicit interface implementation**
- **Default interface methods** (C# 8.0+)

### 2. Generics
- **Type-safe reusable code**
- **Generic constraints**
- **Covariance and contravariance**

### 3. Dependency Injection
- **Built-in DI container**
- **Service lifetime management**
- **Constructor injection**

### 4. Extension Methods
- **Extend existing types**
- **Fluent APIs**
- **Utility functions**

### 5. LINQ
- **Functional programming approach**
- **Method chaining**
- **Lazy evaluation**

### 6. Async/Await
- **Asynchronous programming**
- **Task-based operations**
- **Non-blocking I/O**

---

## การทดสอบ SOLID Principles

### 1. Unit Testing
```csharp
[TestMethod]
public void UserService_CreateUser_ShouldSucceed()
{
    // Arrange
    var mockRepository = new Mock<IUserRepository>();
    var userService = new UserService(mockRepository.Object);
    
    // Act & Assert
    // ทดสอบได้ง่ายเพราะใช้ interface
}
```

### 2. Integration Testing
```csharp
[TestMethod]
public void PaymentProcessor_WithDifferentMethods_ShouldWork()
{
    // ทดสอบกับ implementation จริง
    var processor = new PaymentProcessor();
    var creditCard = new CreditCardPayment();
    var paypal = new PayPalPayment();
    
    // ทั้งสองควรทำงานได้
}
```

---

## Best Practices ใน C# .NET

### 1. Naming Conventions
- **Interface**: ขึ้นต้นด้วย `I` (เช่น `IUserRepository`)
- **Abstract class**: ขึ้นต้นด้วย `Base` หรือ `Abstract`
- **Implementation**: ชื่อที่สื่อความหมาย (เช่น `SqlServerDatabase`)

### 2. Dependency Injection
```csharp
// ใน Startup.cs หรือ Program.cs
services.AddScoped<IUserRepository, SqlServerDatabase>();
services.AddScoped<IEmailService, SmtpEmailService>();
services.AddScoped<UserService>();
```

### 3. Error Handling
```csharp
public class Result<T>
{
    public bool IsSuccess { get; set; }
    public T Data { get; set; }
    public string Error { get; set; }
}
```

### 4. Configuration
```csharp
public class DatabaseSettings
{
    public string ConnectionString { get; set; }
    public int Timeout { get; set; }
}
```

---

## ตัวอย่างการใช้งานจริง

### 1. Web API Controller
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    
    public UsersController(IUserService userService)
    {
        _userService = userService;
    }
    
    [HttpPost]
    public async Task<IActionResult> CreateUser(CreateUserRequest request)
    {
        var result = await _userService.CreateUserAsync(request);
        return Ok(result);
    }
}
```

### 2. Background Service
```csharp
public class EmailBackgroundService : BackgroundService
{
    private readonly IEmailService _emailService;
    
    public EmailBackgroundService(IEmailService emailService)
    {
        _emailService = emailService;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Process emails in background
    }
}
```

---

## สรุป

**SOLID Principles ใน C# .NET ช่วยให้:**
- โค้ดอ่านง่ายและเข้าใจง่าย
- ทดสอบได้ง่าย
- แก้ไขและขยายได้ง่าย
- ลดการพึ่งพาระหว่างส่วนต่างๆ
- เพิ่มความสามารถในการนำกลับใช้

**เมื่อไหร่ควรใช้:**
- เมื่อออกแบบ architecture ใหม่
- เมื่อ refactor โค้ดเก่า
- เมื่อต้องการเพิ่มฟีเจอร์ใหม่
- เมื่อต้องการปรับปรุงคุณภาพโค้ด

---

## อ้างอิง

- [01-single-responsibility-principle.md](./01-single-responsibility-principle.md)
- [02-open-closed-principle.md](./02-open-closed-principle.md)
- [03-liskov-substitution-principle.md](./03-liskov-substitution-principle.md)
- [04-interface-segregation-principle.md](./04-interface-segregation-principle.md)
- [05-dependency-inversion-principle.md](./05-dependency-inversion-principle.md) 