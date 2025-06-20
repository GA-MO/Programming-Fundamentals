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
}

// High-level module - User Service (ขึ้นต่อ concrete class)
public class UserService
{
    private readonly SqlServerDatabase _database;

    public UserService()
    {
        _database = new SqlServerDatabase();
    }

    public async Task CreateUserAsync(string name, string email)
    {
        await _database.SaveUserAsync(name, email);
    }
}

// --- การใช้งาน ---
public class Program
{
    public static async Task Main()
    {
        var userService = new UserService();
        await userService.CreateUserAsync("John Doe", "john@example.com");
    }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **ขึ้นต่อ concrete class**: `UserService` ขึ้นต่อ `SqlServerDatabase` โดยตรง
2. **ยากต่อการทดสอบ**: ไม่สามารถ mock database ได้ง่าย
3. **ยากต่อการเปลี่ยน**: ถ้าต้องการเปลี่ยนเป็น `MongoDB` ต้องแก้ไข `UserService`
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
}

// Low-level module - SQL Server implementation
public class SqlServerRepository : IUserRepository
{
    public async Task SaveUserAsync(string name, string email)
    {
        Console.WriteLine($"Saving user to SQL Server: {name} - {email}");
        await Task.Delay(100);
    }
}

// Low-level module - MongoDB implementation
public class MongoRepository : IUserRepository
{
    public async Task SaveUserAsync(string name, string email)
    {
        Console.WriteLine($"Saving user to MongoDB: {name} - {email}");
        await Task.Delay(100);
    }
}

// High-level module - User Service (ขึ้นต่อ abstraction)
public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }

    public async Task CreateUserAsync(string name, string email)
    {
        await _repository.SaveUserAsync(name, email);
    }
}
```

### ข้อดีของโค้ดที่ดี:
1. **ขึ้นต่อ abstraction**: `UserService` ขึ้นต่อ interface ไม่ใช่ concrete class
2. **ทดสอบง่าย**: สามารถ mock `IUserRepository` ได้ง่าย
3. **เปลี่ยนได้ง่าย**: สามารถเปลี่ยนไปใช้ `MongoRepository` โดยไม่ต้องแก้ไข `UserService`
4. **รักษาหลักการ**: High-level module ไม่ขึ้นต่อ low-level module

---

## ตัวอย่างการใช้งานจริง

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        // ใช้ SQL Server
        Console.WriteLine("Using SQL Server Repository:");
        IUserRepository sqlRepository = new SqlServerRepository();
        var userService1 = new UserService(sqlRepository);
        await userService1.CreateUserAsync("John Doe", "john.sql@example.com");

        Console.WriteLine("\n" + new string('-', 50) + "\n");

        // เปลี่ยนเป็น MongoDB โดยไม่แก้ไข UserService
        Console.WriteLine("Using MongoDB Repository:");
        IUserRepository mongoRepository = new MongoRepository();
        var userService2 = new UserService(mongoRepository);
        await userService2.CreateUserAsync("Jane Doe", "jane.mongo@example.com");
    }
}
```

---

## Dependency Injection Container

ใน .NET เรามักใช้ DI Container ที่มาพร้อมกับ Framework เช่น `Microsoft.Extensions.DependencyInjection`

```csharp
// ตัวอย่างการตั้งค่าในไฟล์ Program.cs หรือ Startup.cs ของ ASP.NET Core
public void ConfigureServices(IServiceCollection services)
{
    // Register IUserRepository ให้ชี้ไปที่ SqlServerRepository
    // หากต้องการเปลี่ยนเป็น MongoRepository ก็แค่เปลี่ยนบรรทัดนี้
    services.AddScoped<IUserRepository, SqlServerRepository>();

    // Register UserService
    services.AddScoped<UserService>();
}

// เวลาใช้งาน Controller หรือ Service อื่นๆ .NET จะฉีด (inject) dependency ให้เอง
public class UserController : ControllerBase
{
    private readonly UserService _userService;

    public UserController(UserService userService)
    {
        _userService = userService;
    }

    [HttpPost]
    public async Task<IActionResult> CreateUser(string name, string email)
    {
        await _userService.CreateUserAsync(name, email);
        return Ok();
    }
}
```

---

## การทดสอบ (Testing)

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using Moq;
using System.Threading.Tasks;

[TestClass]
public class DependencyInversionTests
{
    [TestMethod]
    public async Task UserService_CreateUser_ShouldCallSaveUserAsync()
    {
        // Arrange
        var mockRepository = new Mock<IUserRepository>();
        var userService = new UserService(mockRepository.Object);

        var name = "John Doe";
        var email = "john@example.com";

        // Act
        await userService.CreateUserAsync(name, email);

        // Assert
        // ตรวจสอบว่า SaveUserAsync ถูกเรียก 1 ครั้งด้วยพารามิเตอร์ที่ถูกต้อง
        mockRepository.Verify(r => r.SaveUserAsync(name, email), Times.Once);
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