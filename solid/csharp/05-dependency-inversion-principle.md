# Dependency Inversion Principle (DIP) ใน C# .NET

## ความหมาย
**"ขึ้นต่อ abstraction ไม่ใช่ concrete"** - โปรแกรมควรขึ้นต่อ interface หรือ abstract class แทนที่จะขึ้นต่อ concrete class

## หลักการ
- **High-level modules** ไม่ควรขึ้นต่อ **low-level modules**
- ทั้งสองควรขึ้นต่อ **abstraction**
- **Abstraction** ไม่ควรขึ้นต่อ **details**
- **Details** ควรขึ้นต่อ **abstraction**

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```csharp
// ระบบจัดการผู้ใช้ที่ขึ้นต่อ concrete class
using System;
using System.Threading.Tasks;

// Low-level module - Database
public class SqlServerDatabase
{
    public async Task SaveUserAsync(string name, string email)
    {
        Console.WriteLine($"Saving user to SQL Server: {name} - {email}");
        await Task.Delay(100); // Simulate database operation
    }

    public async Task<string> GetUserAsync(string email)
    {
        Console.WriteLine($"Getting user from SQL Server: {email}");
        await Task.Delay(100);
        return $"User: {email}";
    }
}

// Low-level module - Email Service
public class SmtpEmailService
{
    public async Task SendEmailAsync(string to, string subject, string body)
    {
        Console.WriteLine($"Sending email via SMTP to {to}: {subject}");
        await Task.Delay(100); // Simulate email sending
    }
}

// High-level module - User Service (ขึ้นต่อ concrete class)
public class UserService
{
    private readonly SqlServerDatabase _database;
    private readonly SmtpEmailService _emailService;

    public UserService()
    {
        _database = new SqlServerDatabase();
        _emailService = new SmtpEmailService();
    }

    public async Task CreateUserAsync(string name, string email)
    {
        // บันทึกผู้ใช้ลงฐานข้อมูล
        await _database.SaveUserAsync(name, email);

        // ส่งอีเมลยืนยัน
        await _emailService.SendEmailAsync(email, "Welcome", $"Welcome {name}!");
    }

    public async Task<string> GetUserAsync(string email)
    {
        return await _database.GetUserAsync(email);
    }
}

// Application ที่ขึ้นต่อ UserService
public class Application
{
    private readonly UserService _userService;

    public Application()
    {
        _userService = new UserService();
    }

    public async Task RunAsync()
    {
        await _userService.CreateUserAsync("John Doe", "john@example.com");
        var user = await _userService.GetUserAsync("john@example.com");
        Console.WriteLine(user);
    }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **ขึ้นต่อ concrete class**: UserService ขึ้นต่อ SqlServerDatabase และ SmtpEmailService โดยตรง
2. **ยากต่อการทดสอบ**: ไม่สามารถ mock database หรือ email service ได้
3. **ยากต่อการเปลี่ยน**: ถ้าต้องการเปลี่ยนเป็น MongoDB หรือ SendGrid ต้องแก้ไข UserService
4. **ผิดหลักการ**: High-level module ขึ้นต่อ low-level module

---

## ✅ ตัวอย่างที่ดี (Good Example)

```csharp
using System;
using System.Threading.Tasks;

// Abstraction - Interface สำหรับ Database
public interface IUserRepository
{
    Task SaveUserAsync(string name, string email);
    Task<string> GetUserAsync(string email);
}

// Abstraction - Interface สำหรับ Email Service
public interface IEmailService
{
    Task SendEmailAsync(string to, string subject, string body);
}

// Low-level module - SQL Server implementation
public class SqlServerDatabase : IUserRepository
{
    public async Task SaveUserAsync(string name, string email)
    {
        Console.WriteLine($"Saving user to SQL Server: {name} - {email}");
        await Task.Delay(100);
    }

    public async Task<string> GetUserAsync(string email)
    {
        Console.WriteLine($"Getting user from SQL Server: {email}");
        await Task.Delay(100);
        return $"User from SQL Server: {email}";
    }
}

// Low-level module - MongoDB implementation
public class MongoDatabase : IUserRepository
{
    public async Task SaveUserAsync(string name, string email)
    {
        Console.WriteLine($"Saving user to MongoDB: {name} - {email}");
        await Task.Delay(100);
    }

    public async Task<string> GetUserAsync(string email)
    {
        Console.WriteLine($"Getting user from MongoDB: {email}");
        await Task.Delay(100);
        return $"User from MongoDB: {email}";
    }
}

// Low-level module - SMTP implementation
public class SmtpEmailService : IEmailService
{
    public async Task SendEmailAsync(string to, string subject, string body)
    {
        Console.WriteLine($"Sending email via SMTP to {to}: {subject}");
        await Task.Delay(100);
    }
}

// Low-level module - SendGrid implementation
public class SendGridEmailService : IEmailService
{
    public async Task SendEmailAsync(string to, string subject, string body)
    {
        Console.WriteLine($"Sending email via SendGrid to {to}: {subject}");
        await Task.Delay(100);
    }
}

// High-level module - User Service (ขึ้นต่อ abstraction)
public class UserService
{
    private readonly IUserRepository _repository;
    private readonly IEmailService _emailService;

    public UserService(IUserRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }

    public async Task CreateUserAsync(string name, string email)
    {
        // บันทึกผู้ใช้ลงฐานข้อมูล
        await _repository.SaveUserAsync(name, email);

        // ส่งอีเมลยืนยัน
        await _emailService.SendEmailAsync(email, "Welcome", $"Welcome {name}!");
    }

    public async Task<string> GetUserAsync(string email)
    {
        return await _repository.GetUserAsync(email);
    }
}

// Application ที่ขึ้นต่อ abstraction
public class Application
{
    private readonly UserService _userService;

    public Application(UserService userService)
    {
        _userService = userService;
    }

    public async Task RunAsync()
    {
        await _userService.CreateUserAsync("John Doe", "john@example.com");
        var user = await _userService.GetUserAsync("john@example.com");
        Console.WriteLine(user);
    }
}
```

### ข้อดีของโค้ดที่ดี:
1. **ขึ้นต่อ abstraction**: UserService ขึ้นต่อ interface ไม่ใช่ concrete class
2. **ทดสอบง่าย**: สามารถ mock interface ได้ง่าย
3. **เปลี่ยนได้ง่าย**: สามารถเปลี่ยน implementation โดยไม่แก้ไข UserService
4. **รักษาหลักการ**: High-level module ไม่ขึ้นต่อ low-level module

---

## ตัวอย่างการใช้งานจริง

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        // ใช้ SQL Server และ SMTP
        Console.WriteLine("Using SQL Server and SMTP:");
        var sqlRepository = new SqlServerDatabase();
        var smtpEmail = new SmtpEmailService();
        var userService1 = new UserService(sqlRepository, smtpEmail);
        var app1 = new Application(userService1);
        await app1.RunAsync();

        Console.WriteLine("\n" + new string('-', 50) + "\n");

        // เปลี่ยนเป็น MongoDB และ SendGrid โดยไม่แก้ไข UserService
        Console.WriteLine("Using MongoDB and SendGrid:");
        var mongoRepository = new MongoDatabase();
        var sendGridEmail = new SendGridEmailService();
        var userService2 = new UserService(mongoRepository, sendGridEmail);
        var app2 = new Application(userService2);
        await app2.RunAsync();
    }
}
```

---

## Dependency Injection Container

```csharp
// ตัวอย่างการใช้ Dependency Injection Container
public class ServiceContainer
{
    private readonly Dictionary<Type, object> _services = new();

    public void Register<T>(T implementation)
    {
        _services[typeof(T)] = implementation;
    }

    public T Resolve<T>()
    {
        return (T)_services[typeof(T)];
    }
}

// การใช้งาน DI Container
public class ProgramWithDI
{
    static async Task Main(string[] args)
    {
        // Setup DI Container
        var container = new ServiceContainer();
        container.Register<IUserRepository>(new SqlServerDatabase());
        container.Register<IEmailService>(new SmtpEmailService());

        // Resolve dependencies
        var repository = container.Resolve<IUserRepository>();
        var emailService = container.Resolve<IEmailService>();
        var userService = new UserService(repository, emailService);
        var app = new Application(userService);

        await app.RunAsync();
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
public class DependencyInversionTests
{
    [TestMethod]
    public async Task UserService_CreateUser_ShouldSucceed()
    {
        // Arrange
        var mockRepository = new Mock<IUserRepository>();
        var mockEmailService = new Mock<IEmailService>();
        var userService = new UserService(mockRepository.Object, mockEmailService.Object);

        // Act
        await userService.CreateUserAsync("John", "john@example.com");

        // Assert
        mockRepository.Verify(r => r.SaveUserAsync("John", "john@example.com"), Times.Once);
        mockEmailService.Verify(e => e.SendEmailAsync("john@example.com", "Welcome", "Welcome John!"), Times.Once);
    }

    [TestMethod]
    public async Task UserService_GetUser_ShouldReturnUser()
    {
        // Arrange
        var mockRepository = new Mock<IUserRepository>();
        var mockEmailService = new Mock<IEmailService>();
        var userService = new UserService(mockRepository.Object, mockEmailService.Object);

        mockRepository.Setup(r => r.GetUserAsync("john@example.com"))
                     .ReturnsAsync("User: john@example.com");

        // Act
        var result = await userService.GetUserAsync("john@example.com");

        // Assert
        Assert.AreEqual("User: john@example.com", result);
    }

    [TestMethod]
    public async Task SqlServerDatabase_ShouldImplementInterface()
    {
        // Arrange
        IUserRepository repository = new SqlServerDatabase();

        // Act & Assert
        await repository.SaveUserAsync("John", "john@example.com"); // Should not throw
        var user = await repository.GetUserAsync("john@example.com");
        Assert.IsNotNull(user);
    }

    [TestMethod]
    public async Task MongoDatabase_ShouldImplementInterface()
    {
        // Arrange
        IUserRepository repository = new MongoDatabase();

        // Act & Assert
        await repository.SaveUserAsync("John", "john@example.com"); // Should not throw
        var user = await repository.GetUserAsync("john@example.com");
        Assert.IsNotNull(user);
    }
}
```

---

## สรุป

**DIP ใน C# .NET ช่วยให้:**
- โปรแกรมขึ้นต่อ abstraction ไม่ใช่ concrete
- ง่ายต่อการทดสอบด้วย mock
- ง่ายต่อการเปลี่ยน implementation
- ลดการพึ่งพาระหว่าง module

**เมื่อไหร่ควรใช้:**
- เมื่อต้องการความยืดหยุ่นในการเปลี่ยน implementation
- เมื่อต้องการทดสอบได้ง่าย
- เมื่อต้องการลดการพึ่งพาระหว่างส่วนต่างๆ

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - DIP ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Open/Closed Principle](./02-open-closed-principle.md)** - DIP ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การใช้ interface เล็กๆ ช่วยให้ DIP ยืดหยุ่นขึ้น
``` 