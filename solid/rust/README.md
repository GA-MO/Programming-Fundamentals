# SOLID Design Principles ใน Rust

เอกสารนี้แสดงตัวอย่างการใช้งาน SOLID Design Principles ในภาษา Rust พร้อมตัวอย่างที่ดีและไม่ดี

## หลักการทั้ง 5 ข้อ

1. **[Single Responsibility Principle (SRP)](./01-single-responsibility-principle.md)** - หนึ่ง struct/trait ควรมีหน้าที่เดียว
2. **[Open/Closed Principle (OCP)](./02-open-closed-principle.md)** - เปิดให้ขยาย ปิดให้แก้ไข
3. **[Liskov Substitution Principle (LSP)](./03-liskov-substitution-principle.md)** - ใช้ subtype แทน base type ได้
4. **[Interface Segregation Principle (ISP)](./04-interface-segregation-principle.md)** - แยก trait ให้เล็กลง
5. **[Dependency Inversion Principle (DIP)](./05-dependency-inversion-principle.md)** - ขึ้นต่อ abstraction ไม่ใช่ concrete

## คุณสมบัติพิเศษของ Rust

- **Ownership และ Borrowing**: ช่วยให้ memory safety โดยไม่ต้องใช้ garbage collector
- **Traits**: ระบบ interface ที่ทรงพลัง
- **Pattern Matching**: ช่วยให้ code อ่านง่ายและปลอดภัย
- **Error Handling**: ใช้ Result<T, E> แทน exceptions
- **Zero-cost Abstractions**: abstraction ไม่มี runtime overhead

## การใช้งาน

แต่ละไฟล์จะมีตัวอย่างทั้งแบบที่ดีและไม่ดี พร้อมคำอธิบายและ test cases

---

**หมายเหตุ**: ตัวอย่างทั้งหมดเขียนด้วยภาษา Rust และเน้นการใช้งานจริง 