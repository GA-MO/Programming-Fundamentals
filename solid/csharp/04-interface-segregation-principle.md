# Interface Segregation Principle (ISP) ใน C# .NET

## ความหมาย
**"Client ไม่ควรถูกบังคับให้ขึ้นต่อ interface ที่ไม่ใช้"** - แยก interface ให้เล็กลง แทนที่จะมี interface ใหญ่ที่รวมทุกอย่าง

## หลักการ
- แยก interface ให้เล็กลงตามหน้าที่
- Client ควรขึ้นต่อเฉพาะ method ที่ใช้จริง
- หลีกเลี่ยง "fat interface" ที่มี method มากเกินไป

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```csharp
// Interface ใหญ่ที่รวมทุกอย่าง
using System;
using System.Threading.Tasks;

public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void GetPaid();
    void TakeVacation();
    void AttendMeeting();
    void WriteCode();
    void DesignSystem();
    void TestCode();
    void DeployCode();
    void MonitorSystem();
    void GenerateReport();
}

// Employee ที่ต้อง implement ทุก method
public class Employee : IWorker
{
    public void Work() => Console.WriteLine("Employee is working");
    public void Eat() => Console.WriteLine("Employee is eating");
    public void Sleep() => Console.WriteLine("Employee is sleeping");
    public void GetPaid() => Console.WriteLine("Employee gets paid");
    public void TakeVacation() => Console.WriteLine("Employee takes vacation");
    public void AttendMeeting() => Console.WriteLine("Employee attends meeting");
    public void WriteCode() => Console.WriteLine("Employee writes code");
    public void DesignSystem() => Console.WriteLine("Employee designs system");
    public void TestCode() => Console.WriteLine("Employee tests code");
    public void DeployCode() => Console.WriteLine("Employee deploys code");
    public void MonitorSystem() => Console.WriteLine("Employee monitors system");
    public void GenerateReport() => Console.WriteLine("Employee generates report");
}

// Robot ที่ไม่ต้องการหลาย method แต่ต้อง implement ทั้งหมด
public class Robot : IWorker
{
    public void Work() => Console.WriteLine("Robot is working");
    public void Eat() => throw new NotImplementedException("Robots don't eat");
    public void Sleep() => throw new NotImplementedException("Robots don't sleep");
    public void GetPaid() => throw new NotImplementedException("Robots don't get paid");
    public void TakeVacation() => throw new NotImplementedException("Robots don't take vacation");
    public void AttendMeeting() => throw new NotImplementedException("Robots don't attend meetings");
    public void WriteCode() => Console.WriteLine("Robot writes code");
    public void DesignSystem() => Console.WriteLine("Robot designs system");
    public void TestCode() => Console.WriteLine("Robot tests code");
    public void DeployCode() => Console.WriteLine("Robot deploys code");
    public void MonitorSystem() => Console.WriteLine("Robot monitors system");
    public void GenerateReport() => Console.WriteLine("Robot generates report");
}

// Manager ที่ใช้เฉพาะบาง method
public class Manager
{
    public void ManageWorker(IWorker worker)
    {
        worker.Work();
        worker.AttendMeeting();
        worker.GenerateReport();
        // แต่ manager ไม่สามารถเรียก worker.Eat() หรือ worker.Sleep() ได้
    }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **Interface ใหญ่เกินไป**: IWorker มี method มากเกินไป
2. **Robot ต้อง implement method ที่ไม่ใช้**: ทำให้เกิด NotImplementedException
3. **ยากต่อการขยาย**: ถ้าเพิ่ม method ใหม่ ทุก class ต้อง implement
4. **ผิดหลักการ**: Client ถูกบังคับให้ขึ้นต่อ method ที่ไม่ใช้

---

## ✅ ตัวอย่างที่ดี (Good Example)

```csharp
using System;
using System.Threading.Tasks;

// แยก interface ตามหน้าที่
public interface IWorkable
{
    void Work();
}

public interface IEatable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public interface IPayable
{
    void GetPaid();
}

public interface IVacationable
{
    void TakeVacation();
}

public interface IMeetingAttendable
{
    void AttendMeeting();
}

public interface ICodeWritable
{
    void WriteCode();
}

public interface ISystemDesignable
{
    void DesignSystem();
}

public interface ICodeTestable
{
    void TestCode();
}

public interface ICodeDeployable
{
    void DeployCode();
}

public interface ISystemMonitorable
{
    void MonitorSystem();
}

public interface IReportGeneratable
{
    void GenerateReport();
}

// Employee ที่ implement interface ที่ต้องการ
public class Employee : IWorkable, IEatable, ISleepable, IPayable, 
                       IVacationable, IMeetingAttendable, ICodeWritable,
                       ISystemDesignable, ICodeTestable, ICodeDeployable,
                       ISystemMonitorable, IReportGeneratable
{
    public void Work() => Console.WriteLine("Employee is working");
    public void Eat() => Console.WriteLine("Employee is eating");
    public void Sleep() => Console.WriteLine("Employee is sleeping");
    public void GetPaid() => Console.WriteLine("Employee gets paid");
    public void TakeVacation() => Console.WriteLine("Employee takes vacation");
    public void AttendMeeting() => Console.WriteLine("Employee attends meeting");
    public void WriteCode() => Console.WriteLine("Employee writes code");
    public void DesignSystem() => Console.WriteLine("Employee designs system");
    public void TestCode() => Console.WriteLine("Employee tests code");
    public void DeployCode() => Console.WriteLine("Employee deploys code");
    public void MonitorSystem() => Console.WriteLine("Employee monitors system");
    public void GenerateReport() => Console.WriteLine("Employee generates report");
}

// Robot ที่ implement เฉพาะ interface ที่ต้องการ
public class Robot : IWorkable, ICodeWritable, ISystemDesignable, 
                    ICodeTestable, ICodeDeployable, ISystemMonitorable, 
                    IReportGeneratable
{
    public void Work() => Console.WriteLine("Robot is working");
    public void WriteCode() => Console.WriteLine("Robot writes code");
    public void DesignSystem() => Console.WriteLine("Robot designs system");
    public void TestCode() => Console.WriteLine("Robot tests code");
    public void DeployCode() => Console.WriteLine("Robot deploys code");
    public void MonitorSystem() => Console.WriteLine("Robot monitors system");
    public void GenerateReport() => Console.WriteLine("Robot generates report");
}

// Manager ที่ใช้เฉพาะ interface ที่ต้องการ
public class Manager
{
    public void ManageWorker(IWorkable worker, IMeetingAttendable attendable, IReportGeneratable reportable)
    {
        worker.Work();
        attendable.AttendMeeting();
        reportable.GenerateReport();
    }
}

// HR ที่จัดการเรื่องการเงินและวันหยุด
public class HR
{
    public void ProcessPayroll(IPayable payable)
    {
        payable.GetPaid();
    }

    public void ProcessVacation(IVacationable vacationable)
    {
        vacationable.TakeVacation();
    }
}
```

### ข้อดีของโค้ดที่ดี:
1. **Interface เล็กและเฉพาะเจาะจง**: แต่ละ interface มีหน้าที่เดียว
2. **Client ขึ้นต่อเฉพาะสิ่งที่ใช้**: ไม่ต้อง implement method ที่ไม่ใช้
3. **ขยายง่าย**: เพิ่ม interface ใหม่โดยไม่กระทบส่วนอื่น
4. **ยืดหยุ่น**: สามารถผสมผสาน interface ได้ตามต้องการ

---

## ตัวอย่างการใช้งานจริง

```csharp
class Program
{
    static void Main(string[] args)
    {
        // สร้าง Employee และ Robot
        var employee = new Employee();
        var robot = new Robot();

        // Manager จัดการ worker
        var manager = new Manager();
        Console.WriteLine("Managing Employee:");
        manager.ManageWorker(employee, employee, employee);

        Console.WriteLine("\nManaging Robot:");
        manager.ManageWorker(robot, robot, robot);

        // HR จัดการเรื่องการเงิน
        var hr = new HR();
        Console.WriteLine("\nHR processing payroll:");
        hr.ProcessPayroll(employee);
        // hr.ProcessPayroll(robot); // ไม่สามารถทำได้ เพราะ Robot ไม่ implement IPayable

        Console.WriteLine("\nHR processing vacation:");
        hr.ProcessVacation(employee);
        // hr.ProcessVacation(robot); // ไม่สามารถทำได้ เพราะ Robot ไม่ implement IVacationable
    }
}
```

---

## ตัวอย่างเพิ่มเติม: ระบบจัดการเอกสาร

```csharp
// ระบบจัดการเอกสารที่ใช้ ISP
public interface IReadable
{
    string Read();
}

public interface IWritable
{
    void Write(string content);
}

public interface IPrintable
{
    void Print();
}

public interface IScannable
{
    void Scan();
}

public interface IFaxable
{
    void Fax(string recipient);
}

// Document ที่อ่านและเขียนได้
public class Document : IReadable, IWritable
{
    private string _content = "";

    public string Read() => _content;
    public void Write(string content) => _content = content;
}

// Printer ที่พิมพ์ได้
public class Printer : IPrintable
{
    public void Print() => Console.WriteLine("Printing document...");
}

// Scanner ที่สแกนได้
public class Scanner : IScannable
{
    public void Scan() => Console.WriteLine("Scanning document...");
}

// Fax Machine ที่ส่งแฟกซ์ได้
public class FaxMachine : IFaxable
{
    public void Fax(string recipient) => Console.WriteLine($"Faxing to {recipient}...");
}

// All-in-One ที่ทำได้ทุกอย่าง
public class AllInOnePrinter : IPrintable, IScannable, IFaxable
{
    public void Print() => Console.WriteLine("All-in-one printing...");
    public void Scan() => Console.WriteLine("All-in-one scanning...");
    public void Fax(string recipient) => Console.WriteLine($"All-in-one faxing to {recipient}...");
}
```

---

## การทดสอบ (Testing)

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;
using System;

[TestClass]
public class InterfaceSegregationTests
{
    [TestMethod]
    public void Employee_ShouldImplementAllInterfaces()
    {
        // Arrange
        var employee = new Employee();

        // Act & Assert
        Assert.IsTrue(employee is IWorkable);
        Assert.IsTrue(employee is IEatable);
        Assert.IsTrue(employee is ISleepable);
        Assert.IsTrue(employee is IPayable);
    }

    [TestMethod]
    public void Robot_ShouldNotImplementBiologicalInterfaces()
    {
        // Arrange
        var robot = new Robot();

        // Act & Assert
        Assert.IsTrue(robot is IWorkable);
        Assert.IsFalse(robot is IEatable);
        Assert.IsFalse(robot is ISleepable);
        Assert.IsFalse(robot is IPayable);
    }

    [TestMethod]
    public void Manager_ShouldManageWorkers()
    {
        // Arrange
        var manager = new Manager();
        var employee = new Employee();

        // Act & Assert
        manager.ManageWorker(employee, employee, employee); // Should not throw
    }

    [TestMethod]
    public void HR_ShouldProcessPayroll()
    {
        // Arrange
        var hr = new HR();
        var employee = new Employee();

        // Act & Assert
        hr.ProcessPayroll(employee); // Should not throw
    }

    [TestMethod]
    public void Document_ShouldBeReadableAndWritable()
    {
        // Arrange
        var document = new Document();
        var content = "Hello World";

        // Act
        document.Write(content);
        var result = document.Read();

        // Assert
        Assert.AreEqual(content, result);
    }
}
```

---

## สรุป

**ISP ใน C# .NET ช่วยให้:**
- Interface เล็กและเฉพาะเจาะจง
- Client ขึ้นต่อเฉพาะสิ่งที่ใช้จริง
- ลดการพึ่งพาที่ไม่จำเป็น
- เพิ่มความยืดหยุ่นในการใช้งาน

**เมื่อไหร่ควรใช้:**
- เมื่อ interface มี method มากเกินไป
- เมื่อ client ถูกบังคับให้ implement method ที่ไม่ใช้
- เมื่อต้องการความยืดหยุ่นในการผสมผสาน interface

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - ISP ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface เล็กๆ ช่วยให้ DIP ยืดหยุ่นขึ้น
</rewritten_file> 