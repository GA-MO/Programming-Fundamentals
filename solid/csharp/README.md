# SOLID Design Principles ใน C# .NET

เอกสารนี้แสดงตัวอย่างการใช้งาน SOLID Design Principles ในภาษา C# .NET พร้อมตัวอย่างที่ดีและไม่ดี

## หลักการทั้ง 5 ข้อ

1. **[Single Responsibility Principle (SRP)](./01-single-responsibility-principle.md)** - หนึ่ง class/interface ควรมีหน้าที่เดียว
2. **[Open/Closed Principle (OCP)](./02-open-closed-principle.md)** - เปิดให้ขยาย ปิดให้แก้ไข
3. **[Liskov Substitution Principle (LSP)](./03-liskov-substitution-principle.md)** - ใช้ subtype แทน base type ได้
4. **[Interface Segregation Principle (ISP)](./04-interface-segregation-principle.md)** - แยก interface ให้เล็กลง
5. **[Dependency Inversion Principle (DIP)](./05-dependency-inversion-principle.md)** - ขึ้นต่อ abstraction ไม่ใช่ concrete

## คุณสมบัติพิเศษของ C# .NET

- **Interfaces**: ระบบ interface ที่ทรงพลัง
- **Generics**: สร้าง reusable code ที่ type-safe
- **Dependency Injection**: ระบบ DI ที่แข็งแกร่ง
- **LINQ**: Language Integrated Query
- **Async/Await**: การจัดการ asynchronous operations
- **Attributes**: เพิ่ม metadata และ behavior
- **Extension Methods**: ขยาย functionality ของ existing types
- **Pattern Matching**: ตั้งแต่ C# 7.0

## การใช้งาน

แต่ละไฟล์จะมีตัวอย่างทั้งแบบที่ดีและไม่ดี พร้อมคำอธิบายและ test cases

---

**หมายเหตุ**: ตัวอย่างทั้งหมดเขียนด้วยภาษา C# .NET และเน้นการใช้งานจริง 