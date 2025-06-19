# SOLID Design Principles ใน Go

SOLID เป็นหลักการออกแบบซอฟต์แวร์ที่ช่วยให้โค้ดมีความยืดหยุ่น ดูแลรักษาง่าย และขยายได้ โดยเฉพาะในภาษา Go

## หลักการทั้ง 5 ข้อ

1. **[Single Responsibility Principle (SRP)](./golang/01-single-responsibility-principle.md)** - หนึ่งคลาสควรมีหน้าที่เดียว
2. **[Open/Closed Principle (OCP)](./golang/02-open-closed-principle.md)** - เปิดให้ขยาย ปิดให้แก้ไข
3. **[Liskov Substitution Principle (LSP)](./golang/03-liskov-substitution-principle.md)** - ใช้ subclass แทน parent class ได้
4. **[Interface Segregation Principle (ISP)](./golang/04-interface-segregation-principle.md)** - แยก interface ให้เล็กลง
5. **[Dependency Inversion Principle (DIP)](./golang/05-dependency-inversion-principle.md)** - ขึ้นต่อ abstraction ไม่ใช่ concrete

## ตัวอย่างการใช้งานจริง

แต่ละหลักการจะมีตัวอย่างทั้งแบบที่ดีและไม่ดี เพื่อให้เข้าใจง่ายสำหรับ Junior Developer

## การอ้างอิงข้ามไฟล์

ไฟล์แต่ละไฟล์จะมีการอ้างอิงไปยังหลักการอื่นๆ ที่เกี่ยวข้อง เพื่อให้เห็นภาพรวมของการใช้งานร่วมกัน

## References - ตัวอย่างในภาษาอื่น

### Rust Implementation
- **[Rust SOLID Principles Overview](./rust/README.md)** - ภาพรวม SOLID Principles ใน Rust
- **[Rust SRP](./rust/01-single-responsibility-principle.md)** - Single Responsibility Principle ใน Rust
- **[Rust OCP](./rust/02-open-closed-principle.md)** - Open/Closed Principle ใน Rust
- **[Rust LSP](./rust/03-liskov-substitution-principle.md)** - Liskov Substitution Principle ใน Rust
- **[Rust ISP](./rust/04-interface-segregation-principle.md)** - Interface Segregation Principle ใน Rust
- **[Rust DIP](./rust/05-dependency-inversion-principle.md)** - Dependency Inversion Principle ใน Rust
- **[Rust Summary](./rust/summary.md)** - สรุป SOLID Principles ใน Rust

### TypeScript Implementation
- **[TypeScript SOLID Principles Overview](./typescript/README.md)** - ภาพรวม SOLID Principles ใน TypeScript
- **[TypeScript SRP](./typescript/01-single-responsibility-principle.md)** - Single Responsibility Principle ใน TypeScript
- **[TypeScript OCP](./typescript/02-open-closed-principle.md)** - Open/Closed Principle ใน TypeScript
- **[TypeScript LSP](./typescript/03-liskov-substitution-principle.md)** - Liskov Substitution Principle ใน TypeScript
- **[TypeScript ISP](./typescript/04-interface-segregation-principle.md)** - Interface Segregation Principle ใน TypeScript
- **[TypeScript DIP](./typescript/05-dependency-inversion-principle.md)** - Dependency Inversion Principle ใน TypeScript
- **[TypeScript Summary](./typescript/summary.md)** - สรุป SOLID Principles ใน TypeScript

### C# .NET Implementation
- **[C# .NET SOLID Principles Overview](./csharp/README.md)** - ภาพรวม SOLID Principles ใน C# .NET
- **[C# .NET SRP](./csharp/01-single-responsibility-principle.md)** - Single Responsibility Principle ใน C# .NET
- **[C# .NET OCP](./csharp/02-open-closed-principle.md)** - Open/Closed Principle ใน C# .NET
- **[C# .NET LSP](./csharp/03-liskov-substitution-principle.md)** - Liskov Substitution Principle ใน C# .NET
- **[C# .NET ISP](./csharp/04-interface-segregation-principle.md)** - Interface Segregation Principle ใน C# .NET
- **[C# .NET DIP](./csharp/05-dependency-inversion-principle.md)** - Dependency Inversion Principle ใน C# .NET
- **[C# .NET Summary](./csharp/summary.md)** - สรุป SOLID Principles ใน C# .NET

## การเปรียบเทียบระหว่างภาษา

| หลักการ | Go | Rust | TypeScript | C# .NET |
|---------|----|------|------------|---------|
| SRP | struct/interface | struct/trait | class/interface | class/interface |
| OCP | interface | trait | interface | interface/abstract class |
| LSP | interface | trait | interface | interface |
| ISP | interface | trait | interface | interface |
| DIP | interface | trait | interface | interface/DI container |

## คุณสมบัติพิเศษของแต่ละภาษา

### Go
- **Interfaces**: Implicit implementation
- **Composition over inheritance**
- **Goroutines และ channels**
- **Static typing**

### Rust
- **Ownership และ Borrowing**
- **Traits system**
- **Pattern Matching**
- **Zero-cost Abstractions**
- **Memory safety**

### TypeScript
- **Static Typing**
- **Interfaces**
- **Generics**
- **Decorators**
- **Union Types**
- **Type Guards**

### C# .NET
- **Interfaces**: Explicit implementation
- **Dependency Injection**: Built-in DI container
- **Generics**: Type-safe reusable code
- **LINQ**: Language Integrated Query
- **Async/Await**: Asynchronous programming
- **Extension Methods**: Extend existing types
- **Attributes**: Metadata and behavior
- **Pattern Matching**: C# 7.0+

---

**หมายเหตุ**: ตัวอย่างทั้งหมดเขียนด้วยภาษา Go และเน้นการใช้งานจริงในชีวิตประจำวัน พร้อม References ไปยังตัวอย่างในภาษา Rust, TypeScript และ C# .NET
