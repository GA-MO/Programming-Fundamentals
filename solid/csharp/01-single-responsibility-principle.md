# Single Responsibility Principle (SRP) ใน C# .NET

## ความหมาย
**"หนึ่ง class/interface ควรมีหน้าที่เดียว"** - หมายความว่าแต่ละ class หรือ method ควรมีเหตุผลเดียวที่จะเปลี่ยนแปลง

## หลักการ
- หนึ่ง class = หนึ่งหน้าที่
- ถ้ามีเหตุผลหลายอย่างที่จะเปลี่ยนแปลง class แสดงว่าผิดหลักการนี้
- ช่วยให้โค้ดอ่านง่าย ทดสอบง่าย และดูแลรักษาง่าย

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```csharp
// UserService ที่ทำหลายหน้าที่เกินไป
using System;
using System.Threading.Tasks;

public class User
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string Password { get; set; }
}

public class UserService
{
    private readonly string _dbConnection;

    public UserService()
    {
        _dbConnection = "Server=localhost;Database=Users;Trusted_Connection=true;";
    }

    public async Task CreateUserAsync(User user)
    {
        // 1. ตรวจสอบข้อมูล
        if (string.IsNullOrEmpty(user.Name) || string.IsNullOrEmpty(user.Email))
        {
            throw new ArgumentException("Name and email are required");
        }

        // 2. เข้ารหัส password
        var hashedPassword = HashPassword(user.Password);

        // 3. บันทึกลงฐานข้อมูล
        await SaveToDatabaseAsync(user.Name, user.Email, hashedPassword);

        // 4. ส่งอีเมลยืนยัน
        await SendWelcomeEmailAsync(user.Email, user.Name);

        // 5. บันทึก log
        Console.WriteLine($"User created: {user.Email}");
    }

    private string HashPassword(string password)
    {
        // โค้ดเข้ารหัส password
        return $"hashed_{password}";
    }

    private async Task SaveToDatabaseAsync(string name, string email, string password)
    {
        Console.WriteLine($"Saving to database: {name} - {email}");
        await Task.Delay(100); // Simulate database operation
    }

    private async Task SendWelcomeEmailAsync(string email, string name)
    {
        Console.WriteLine($"Sending welcome email to {email}: {name}");
        await Task.Delay(100); // Simulate email sending
    }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **หลายหน้าที่**: ตรวจสอบข้อมูล, เข้ารหัส, บันทึก DB, ส่งอีเมล, บันทึก log
2. **ยากต่อการทดสอบ**: ต้อง mock หลายอย่าง
3. **ยากต่อการแก้ไข**: ถ้าเปลี่ยนวิธีส่งอีเมลต้องแก้ไข UserService
4. **ยากต่อการนำกลับใช้**: ไม่สามารถใช้ส่วนการเข้ารหัสแยกได้

---

## ✅ ตัวอย่างที่ดี (Good Example)

```csharp
using System;
using System.Threading.Tasks;

public class User
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string Password { get; set; }
}

// 1. UserValidator - ตรวจสอบข้อมูล
public class UserValidator
{
    public static void Validate(User user)
    {
        if (string.IsNullOrEmpty(user.Name) || string.IsNullOrEmpty(user.Email))
        {
            throw new ArgumentException("Name and email are required");
        }
    }
}

// 2. PasswordHasher - เข้ารหัส password
public class PasswordHasher
{
    public static string Hash(string password)
    {
        return $"hashed_{password}";
    }
}

// 3. UserRepository - จัดการฐานข้อมูล
public interface IUserRepository
{
    Task CreateAsync(User user);
}

public class UserRepository : IUserRepository
{
    private readonly string _connectionString;

    public UserRepository()
    {
        _connectionString = "Server=localhost;Database=Users;Trusted_Connection=true;";
    }

    public async Task CreateAsync(User user)
    {
        Console.WriteLine($"Saving to database: {user.Name} - {user.Email}");
        await Task.Delay(100); // Simulate database operation
    }
}

// 4. EmailService - ส่งอีเมล
public interface IEmailService
{
    Task SendWelcomeEmailAsync(User user);
}

public class EmailService : IEmailService
{
    public async Task SendWelcomeEmailAsync(User user)
    {
        Console.WriteLine($"Sending welcome email to {user.Email}: {user.Name}");
        await Task.Delay(100); // Simulate email sending
    }
}

// 5. Logger - บันทึก log
public interface ILogger
{
    void LogUserCreated(string email);
}

public class ConsoleLogger : ILogger
{
    public void LogUserCreated(string email)
    {
        Console.WriteLine($"User created: {email}");
    }
}

// 6. UserService - ประสานงานระหว่างส่วนต่างๆ
public class UserService
{
    private readonly IUserRepository _repository;
    private readonly IEmailService _emailService;
    private readonly ILogger _logger;

    public UserService(IUserRepository repository, IEmailService emailService, ILogger logger)
    {
        _repository = repository;
        _emailService = emailService;
        _logger = logger;
    }

    public async Task CreateUserAsync(User user)
    {
        // 1. ตรวจสอบข้อมูล
        UserValidator.Validate(user);

        // 2. เข้ารหัส password
        var hashedPassword = PasswordHasher.Hash(user.Password);
        var userWithHash = new User
        {
            Name = user.Name,
            Email = user.Email,
            Password = hashedPassword
        };

        // 3. บันทึกลงฐานข้อมูล
        await _repository.CreateAsync(userWithHash);

        // 4. ส่งอีเมลยืนยัน
        await _emailService.SendWelcomeEmailAsync(user);

        // 5. บันทึก log
        _logger.LogUserCreated(user.Email);
    }
}
```

### ข้อดีของโค้ดที่ดี:
1. **แต่ละ class มีหน้าที่เดียว**: ง่ายต่อการเข้าใจและแก้ไข
2. **ทดสอบง่าย**: สามารถทดสอบแต่ละส่วนแยกกันได้
3. **นำกลับใช้ได้**: สามารถใช้ PasswordHasher ในที่อื่นได้
4. **ขยายง่าย**: ถ้าต้องการเปลี่ยนวิธีส่งอีเมล แค่แก้ไข EmailService
5. **Dependency Injection**: ง่ายต่อการ mock และทดสอบ

---

## ตัวอย่างการใช้งานจริง

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        // Setup dependencies
        var repository = new UserRepository();
        var emailService = new EmailService();
        var logger = new ConsoleLogger();

        var userService = new UserService(repository, emailService, logger);

        var user = new User
        {
            Name = "John Doe",
            Email = "john@example.com",
            Password = "password123"
        };

        await userService.CreateUserAsync(user);
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
public class UserServiceTests
{
    [TestMethod]
    public void UserValidator_ValidUser_ShouldNotThrow()
    {
        // Arrange
        var user = new User
        {
            Name = "John",
            Email = "john@example.com",
            Password = "password"
        };

        // Act & Assert
        UserValidator.Validate(user); // Should not throw
    }

    [TestMethod]
    [ExpectedException(typeof(ArgumentException))]
    public void UserValidator_InvalidUser_ShouldThrow()
    {
        // Arrange
        var user = new User
        {
            Name = "",
            Email = "",
            Password = "password"
        };

        // Act
        UserValidator.Validate(user);
    }

    [TestMethod]
    public void PasswordHasher_ShouldHashPassword()
    {
        // Arrange
        var password = "password123";

        // Act
        var hashed = PasswordHasher.Hash(password);

        // Assert
        Assert.IsTrue(hashed.StartsWith("hashed_"));
    }

    [TestMethod]
    public async Task UserService_CreateUser_ShouldSucceed()
    {
        // Arrange
        var mockRepository = new Mock<IUserRepository>();
        var mockEmailService = new Mock<IEmailService>();
        var mockLogger = new Mock<ILogger>();

        var userService = new UserService(mockRepository.Object, mockEmailService.Object, mockLogger.Object);

        var user = new User
        {
            Name = "John",
            Email = "john@example.com",
            Password = "password123"
        };

        // Act
        await userService.CreateUserAsync(user);

        // Assert
        mockRepository.Verify(r => r.CreateAsync(It.IsAny<User>()), Times.Once);
        mockEmailService.Verify(e => e.SendWelcomeEmailAsync(It.IsAny<User>()), Times.Once);
        mockLogger.Verify(l => l.LogUserCreated(It.IsAny<string>()), Times.Once);
    }
}
```

---

## สรุป

**SRP ใน C# .NET ช่วยให้:**
- โค้ดอ่านง่ายและเข้าใจง่าย
- ทดสอบได้ง่าย
- แก้ไขและขยายได้ง่าย
- ลดการพึ่งพากันระหว่างส่วนต่างๆ
- เพิ่มความสามารถในการนำกลับใช้

**เมื่อไหร่ควรใช้:**
- เมื่อ class หรือ method มีหน้าที่หลายอย่าง
- เมื่อยากต่อการทดสอบ
- เมื่อแก้ไขส่วนหนึ่งกระทบส่วนอื่น

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก interface เล็กๆ ช่วยให้ SRP ชัดเจนขึ้น 