# Liskov Substitution Principle (LSP) ใน C# .NET

## ความหมาย
**"Subtype ต้องสามารถแทนที่ base type ได้โดยไม่ทำให้โปรแกรมผิดพลาด"** - ถ้า S เป็น subtype ของ T แล้ว object ของ S ต้องสามารถแทนที่ object ของ T ได้

## หลักการ
- Subtype ต้องรักษา contract ของ base type
- ไม่ควรทำให้เกิด exception ที่ไม่คาดหวัง
- ต้องรักษา preconditions และ postconditions

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```csharp
// Logger interface ที่มีปัญหา
using System;

public interface ILogger
{
    void Log(string message);
    int GetLogCount();
}

// FileLogger - บันทึกลงไฟล์
public class FileLogger : ILogger
{
    private int logCount = 0;

    public void Log(string message)
    {
        Console.WriteLine($"Writing to file: {message}");
        logCount++;
    }

    public int GetLogCount()
    {
        return logCount;
    }
}

// DatabaseLogger - บันทึกลงฐานข้อมูล
public class DatabaseLogger : ILogger
{
    public void Log(string message)
    {
        Console.WriteLine($"Writing to database: {message}");
    }

    public int GetLogCount()
    {
        // ละเมิด LSP! ไม่สามารถนับจำนวน log ได้
        return -1; // หรือ throw exception!
    }
}

// LogService ที่ใช้งาน
public class LogService
{
    public static void ProcessLogs(ILogger logger, string[] messages)
    {
        foreach (var message in messages)
        {
            logger.Log(message);
        }

        int count = logger.GetLogCount();
        Console.WriteLine($"Total logs written: {count}");
        // ถ้าเป็น DatabaseLogger จะได้ -1 ซึ่งไม่ถูกต้อง
    }
}
```

### ปัญหา:
- **DatabaseLogger ไม่สามารถนับ log ได้** แต่ต้อง implement `GetLogCount()`
- **ผิด contract** ของ ILogger interface
- **ทำให้โปรแกรมผิดพลาด** เมื่อใช้ DatabaseLogger

---

## ✅ ตัวอย่างที่ดี (Good Example)

```csharp
using System;

// แยก interface ตามความสามารถ
public interface ILogger
{
    void Log(string message);
}

public interface ILogCounter
{
    int GetLogCount();
}

// FileLogger - สามารถบันทึกและนับได้
public class FileLogger : ILogger, ILogCounter
{
    private int logCount = 0;

    public void Log(string message)
    {
        Console.WriteLine($"Writing to file: {message}");
        logCount++;
    }

    public int GetLogCount()
    {
        return logCount;
    }
}

// DatabaseLogger - สามารถบันทึกได้อย่างเดียว
public class DatabaseLogger : ILogger
{
    public void Log(string message)
    {
        Console.WriteLine($"Writing to database: {message}");
    }
}

// EmailSender interface
public interface IEmailSender
{
    void Send(string to, string subject, string message);
}

// SMTPSender
public class SMTPSender : IEmailSender
{
    public void Send(string to, string subject, string message)
    {
        Console.WriteLine($"Sending email to {to}: {subject}");
    }
}

// MockSender สำหรับทดสอบ
public class MockSender : IEmailSender
{
    public void Send(string to, string subject, string message)
    {
        Console.WriteLine($"Mock: Email sent to {to}");
        // ไม่ทำอะไรจริง แต่ไม่ error
    }
}

// ฟังก์ชันที่ใช้งาน
public class LoggingService
{
    public static void WriteLog(ILogger logger, string message)
    {
        logger.Log(message);
    }

    public static void CountLogs(ILogCounter counter)
    {
        int count = counter.GetLogCount();
        Console.WriteLine($"Total logs: {count}");
    }
}

public class EmailService
{
    public static void SendNotification(IEmailSender sender, string email, string subject, string message)
    {
        sender.Send(email, subject, message);
    }
}
```

### ข้อดี:
- **รักษา contract** แต่ละ interface ทำหน้าที่เฉพาะ
- **ไม่มีปัญหา** เมื่อแทนที่กัน
- **เข้าใจง่าย** แยกความรับผิดชอบชัดเจน

---

## ตัวอย่างการใช้งาน

```csharp
class Program
{
    static void Main(string[] args)
    {
        var fileLogger = new FileLogger();
        var dbLogger = new DatabaseLogger();
        var smtpSender = new SMTPSender();
        var mockSender = new MockSender();

        // ทุก logger สามารถบันทึกได้
        LoggingService.WriteLog(fileLogger, "File log message");
        LoggingService.WriteLog(dbLogger, "Database log message");

        // เฉพาะ FileLogger เท่านั้นที่นับได้
        LoggingService.CountLogs(fileLogger);
        // LoggingService.CountLogs(dbLogger); // Compile error - ดีแล้ว!

        // ทุก sender สามารถส่งอีเมลได้
        EmailService.SendNotification(smtpSender, "user@example.com", "Welcome", "Hello!");
        EmailService.SendNotification(mockSender, "test@example.com", "Test", "Test message");
    }
}
```

---

## การทดสอบ

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class LiskovSubstitutionTests
{
    [TestMethod]
    public void Loggers_ShouldLogCorrectly()
    {
        // Arrange
        var fileLogger = new FileLogger();
        var dbLogger = new DatabaseLogger();

        // Act & Assert
        Assert.DoesNotThrow(() => fileLogger.Log("test message"));
        Assert.DoesNotThrow(() => dbLogger.Log("test message"));

        // เฉพาะ FileLogger เท่านั้นที่นับได้
        Assert.AreEqual(1, fileLogger.GetLogCount());
    }

    [TestMethod]
    public void EmailSenders_ShouldBeSubstitutable()
    {
        // Arrange
        IEmailSender smtpSender = new SMTPSender();
        IEmailSender mockSender = new MockSender();

        // Act & Assert
        Assert.DoesNotThrow(() => smtpSender.Send("test@example.com", "Test", "Message"));
        Assert.DoesNotThrow(() => mockSender.Send("test@example.com", "Test", "Message"));
    }
}
```

---

## ตัวอย่างเพิ่มเติม: การบันทึกข้อมูล

```csharp
// DataSaver interface
public interface IDataSaver
{
    void Save(string data);
}

// FileSaver
public class FileSaver : IDataSaver
{
    public void Save(string data)
    {
        Console.WriteLine($"Saving to file: {data}");
    }
}

// DatabaseSaver
public class DatabaseSaver : IDataSaver
{
    public void Save(string data)
    {
        Console.WriteLine($"Saving to database: {data}");
    }
}

// MockSaver
public class MockSaver : IDataSaver
{
    public void Save(string data)
    {
        Console.WriteLine($"Mock save: {data}");
        // สำหรับทดสอบ
    }
}

public class DataService
{
    public static void SaveUserData(IDataSaver saver, string userData)
    {
        saver.Save(userData);
    }
}
```

---

## สรุป

**LSP ใน C# .NET ช่วยให้:**
- โค้ดทำงานได้ถูกต้องเมื่อแทนที่ type
- ลดข้อผิดพลาดในการใช้งาน
- ง่ายต่อการทดสอบ
- เพิ่มความยืดหยุ่นของโค้ด

**หลักการสำคัญ:**
- แยก interface ตามความสามารถ
- รักษา contract ของ interface
- ไม่ทำให้เกิดข้อผิดพลาดที่ไม่คาดคิด

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Open/Closed Principle](./02-open-closed-principle.md)** - LSP ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface ช่วยให้ LSP ชัดเจนขึ้น
</rewritten_file> 