# Single Responsibility Principle (SRP) ใน TypeScript

## ความหมาย
**"หนึ่ง class/interface ควรมีหน้าที่เดียว"** - หมายความว่าแต่ละ class หรือ function ควรมีเหตุผลเดียวที่จะเปลี่ยนแปลง

## หลักการ
- หนึ่ง class = หนึ่งหน้าที่
- ถ้ามีเหตุผลหลายอย่างที่จะเปลี่ยนแปลง class แสดงว่าผิดหลักการนี้
- ช่วยให้โค้ดอ่านง่าย ทดสอบง่าย และดูแลรักษาง่าย

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```typescript
// UserService ที่ทำหลายหน้าที่เกินไป
interface User {
  name: string;
  email: string;
  password: string;
}

class UserService {
  private dbConnection: string;

  constructor() {
    this.dbConnection = 'mysql://localhost:3306/users';
  }

  createUser(user: User): Promise<void> {
    // 1. ตรวจสอบข้อมูล
    if (!user.name || !user.email) {
      throw new Error('Name and email are required');
    }

    // 2. เข้ารหัส password
    const hashedPassword = this.hashPassword(user.password);

    // 3. บันทึกลงฐานข้อมูล
    this.saveToDatabase(user.name, user.email, hashedPassword);

    // 4. ส่งอีเมลยืนยัน
    this.sendWelcomeEmail(user.email, user.name);

    // 5. บันทึก log
    console.log(`User created: ${user.email}`);

    return Promise.resolve();
  }

  private hashPassword(password: string): string {
    // โค้ดเข้ารหัส password
    return `hashed_${password}`;
  }

  private saveToDatabase(name: string, email: string, password: string): void {
    console.log(`Saving to database: ${name} - ${email}`);
  }

  private sendWelcomeEmail(email: string, name: string): void {
    console.log(`Sending welcome email to ${email}: ${name}`);
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

```typescript
interface User {
  name: string;
  email: string;
  password: string;
}

// 1. UserValidator - ตรวจสอบข้อมูล
class UserValidator {
  static validate(user: User): void {
    if (!user.name || !user.email) {
      throw new Error('Name and email are required');
    }
  }
}

// 2. PasswordHasher - เข้ารหัส password
class PasswordHasher {
  static hash(password: string): string {
    return `hashed_${password}`;
  }
}

// 3. UserRepository - จัดการฐานข้อมูล
class UserRepository {
  private connection: string;

  constructor() {
    this.connection = 'mysql://localhost:3306/users';
  }

  create(user: User): Promise<void> {
    console.log(`Saving to database: ${user.name} - ${user.email}`);
    return Promise.resolve();
  }
}

// 4. EmailService - ส่งอีเมล
class EmailService {
  static sendWelcomeEmail(user: User): Promise<void> {
    console.log(`Sending welcome email to ${user.email}: ${user.name}`);
    return Promise.resolve();
  }
}

// 5. Logger - บันทึก log
class Logger {
  static logUserCreated(email: string): void {
    console.log(`User created: ${email}`);
  }
}

// 6. UserService - ประสานงานระหว่างส่วนต่างๆ
class UserService {
  private repository: UserRepository;

  constructor() {
    this.repository = new UserRepository();
  }

  async createUser(user: User): Promise<void> {
    // 1. ตรวจสอบข้อมูล
    UserValidator.validate(user);

    // 2. เข้ารหัส password
    const hashedPassword = PasswordHasher.hash(user.password);
    const userWithHash = { ...user, password: hashedPassword };

    // 3. บันทึกลงฐานข้อมูล
    await this.repository.create(userWithHash);

    // 4. ส่งอีเมลยืนยัน
    await EmailService.sendWelcomeEmail(user);

    // 5. บันทึก log
    Logger.logUserCreated(user.email);
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

```typescript
async function main(): Promise<void> {
  const userService = new UserService();

  const user: User = {
    name: 'John Doe',
    email: 'john@example.com',
    password: 'password123'
  };

  await userService.createUser(user);
}

main().catch(console.error);
```

---

## การทดสอบ (Testing)

```typescript
// user-validator.test.ts
describe('UserValidator', () => {
  test('should validate valid user', () => {
    const user: User = {
      name: 'John',
      email: 'john@example.com',
      password: 'password'
    };

    expect(() => UserValidator.validate(user)).not.toThrow();
  });

  test('should throw error for invalid user', () => {
    const user: User = {
      name: '',
      email: '',
      password: 'password'
    };

    expect(() => UserValidator.validate(user)).toThrow('Name and email are required');
  });
});

// password-hasher.test.ts
describe('PasswordHasher', () => {
  test('should hash password', () => {
    const password = 'password123';
    const hashed = PasswordHasher.hash(password);
    
    expect(hashed).toMatch(/^hashed_/);
  });
});

// user-service.test.ts
describe('UserService', () => {
  test('should create user successfully', async () => {
    const userService = new UserService();
    const user: User = {
      name: 'John',
      email: 'john@example.com',
      password: 'password123'
    };

    await expect(userService.createUser(user)).resolves.not.toThrow();
  });
});
```

---

## สรุป

**SRP ใน TypeScript ช่วยให้:**
- โค้ดอ่านง่ายและเข้าใจง่าย
- ทดสอบได้ง่าย
- แก้ไขและขยายได้ง่าย
- ลดการพึ่งพากันระหว่างส่วนต่างๆ
- เพิ่มความสามารถในการนำกลับใช้

**เมื่อไหร่ควรใช้:**
- เมื่อ class หรือ function มีหน้าที่หลายอย่าง
- เมื่อยากต่อการทดสอบ
- เมื่อแก้ไขส่วนหนึ่งกระทบส่วนอื่น

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก interface เล็กๆ ช่วยให้ SRP ชัดเจนขึ้น 