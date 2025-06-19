# SOLID Design Principles ใน TypeScript

เอกสารนี้แสดงตัวอย่างการใช้งาน SOLID Design Principles ในภาษา TypeScript พร้อมตัวอย่างที่ดีและไม่ดี

## หลักการทั้ง 5 ข้อ

1. **[Single Responsibility Principle (SRP)](./01-single-responsibility-principle.md)** - หนึ่ง class/interface ควรมีหน้าที่เดียว
2. **[Open/Closed Principle (OCP)](./02-open-closed-principle.md)** - เปิดให้ขยาย ปิดให้แก้ไข
3. **[Liskov Substitution Principle (LSP)](./03-liskov-substitution-principle.md)** - ใช้ subtype แทน base type ได้
4. **[Interface Segregation Principle (ISP)](./04-interface-segregation-principle.md)** - แยก interface ให้เล็กลง
5. **[Dependency Inversion Principle (DIP)](./05-dependency-inversion-principle.md)** - ขึ้นต่อ abstraction ไม่ใช่ concrete

## คุณสมบัติพิเศษของ TypeScript

- **Static Typing**: ตรวจสอบ type ใน compile time
- **Interfaces**: กำหนด contract ที่ชัดเจน
- **Generics**: สร้าง reusable code ที่ type-safe
- **Decorators**: เพิ่ม metadata และ behavior
- **Union Types**: รวมหลาย type เข้าด้วยกัน
- **Type Guards**: ตรวจสอบ type ใน runtime

## การใช้งาน

แต่ละไฟล์จะมีตัวอย่างทั้งแบบที่ดีและไม่ดี พร้อมคำอธิบายและ test cases

---

**หมายเหตุ**: ตัวอย่างทั้งหมดเขียนด้วยภาษา TypeScript และเน้นการใช้งานจริง 