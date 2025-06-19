# Dependency Inversion Principle (DIP) ใน TypeScript

## ความหมาย
**"ขึ้นต่อ abstraction ไม่ใช่ concrete"** - หมายความว่า high-level modules ไม่ควรขึ้นต่อ low-level modules แต่ควรขึ้นต่อ abstraction

## หลักการ
- High-level modules ไม่ควรขึ้นต่อ low-level modules
- ทั้งสองควรขึ้นต่อ abstraction
- Abstraction ไม่ควรขึ้นต่อ details
- Details ควรขึ้นต่อ abstraction

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```typescript
// High-level module ขึ้นต่อ low-level module โดยตรง
interface User {
  id: number;
  name: string;
  email: string;
}

// Low-level module
class MySQLDatabase {
  private connectionString: string;

  constructor() {
    this.connectionString = 'mysql://localhost:3306/users';
  }

  async saveUser(user: User): Promise<void> {
    console.log(`Saving user to MySQL: ${user.name}`);
  }

  async getUser(id: number): Promise<User> {
    console.log(`Getting user from MySQL: ${id}`);
    return {
      id,
      name: 'John Doe',
      email: 'john@example.com'
    };
  }
}

// High-level module ขึ้นต่อ low-level module โดยตรง
class UserService {
  private database: MySQLDatabase;

  constructor() {
    this.database = new MySQLDatabase();
  }

  async createUser(user: User): Promise<void> {
    await this.database.saveUser(user);
  }

  async getUser(id: number): Promise<User> {
    return await this.database.getUser(id);
  }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **UserService ขึ้นต่อ MySQLDatabase โดยตรง**: ถ้าต้องการเปลี่ยนเป็น PostgreSQL ต้องแก้ไข UserService
2. **ยากต่อการทดสอบ**: ต้องใช้ MySQL จริงในการทดสอบ
3. **ยากต่อการขยาย**: ถ้าต้องการเพิ่ม database ใหม่ต้องแก้ไข UserService
4. **ผิดหลัก DIP**: High-level module ขึ้นต่อ low-level module

---

## ✅ ตัวอย่างที่ดี (Good Example)

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

// Abstraction (interface)
interface UserRepository {
  saveUser(user: User): Promise<void>;
  getUser(id: number): Promise<User>;
}

// Low-level module - MySQL implementation
class MySQLDatabase implements UserRepository {
  private connectionString: string;

  constructor() {
    this.connectionString = 'mysql://localhost:3306/users';
  }

  async saveUser(user: User): Promise<void> {
    console.log(`Saving user to MySQL: ${user.name}`);
  }

  async getUser(id: number): Promise<User> {
    console.log(`Getting user from MySQL: ${id}`);
    return {
      id,
      name: 'John Doe',
      email: 'john@example.com'
    };
  }
}

// Low-level module - PostgreSQL implementation
class PostgreSQLDatabase implements UserRepository {
  private connectionString: string;

  constructor() {
    this.connectionString = 'postgresql://localhost:5432/users';
  }

  async saveUser(user: User): Promise<void> {
    console.log(`Saving user to PostgreSQL: ${user.name}`);
  }

  async getUser(id: number): Promise<User> {
    console.log(`Getting user from PostgreSQL: ${id}`);
    return {
      id,
      name: 'John Doe',
      email: 'john@example.com'
    };
  }
}

// Low-level module - In-memory implementation (for testing)
class InMemoryDatabase implements UserRepository {
  private users: Map<number, User> = new Map();

  async saveUser(user: User): Promise<void> {
    console.log(`Saving user to InMemory: ${user.name}`);
    this.users.set(user.id, user);
  }

  async getUser(id: number): Promise<User> {
    console.log(`Getting user from InMemory: ${id}`);
    const user = this.users.get(id);
    if (!user) {
      throw new Error('User not found');
    }
    return user;
  }
}

// High-level module ขึ้นต่อ abstraction
class UserService {
  private repository: UserRepository;

  constructor(repository: UserRepository) {
    this.repository = repository;
  }

  async createUser(user: User): Promise<void> {
    await this.repository.saveUser(user);
  }

  async getUser(id: number): Promise<User> {
    return await this.repository.getUser(id);
  }
}
```

### ข้อดีของโค้ดที่ดี:
1. **UserService ขึ้นต่อ abstraction**: ไม่ขึ้นต่อ concrete implementation
2. **ทดสอบง่าย**: สามารถใช้ InMemoryDatabase ในการทดสอบ
3. **ขยายง่าย**: เพิ่ม database ใหม่ได้โดยไม่แก้ไข UserService
4. **ยืดหยุ่น**: สามารถเปลี่ยน database ได้ง่าย

---

## ตัวอย่างการใช้งานจริง

```typescript
async function main(): Promise<void> {
  // ใช้ MySQL
  const mysqlDb = new MySQLDatabase();
  const userServiceMySQL = new UserService(mysqlDb);
  
  const user: User = {
    id: 1,
    name: 'John Doe',
    email: 'john@example.com'
  };
  
  await userServiceMySQL.createUser(user);
  const retrievedUser = await userServiceMySQL.getUser(1);
  console.log('Retrieved user:', retrievedUser);

  // เปลี่ยนเป็น PostgreSQL
  const postgresDb = new PostgreSQLDatabase();
  const userServicePostgres = new UserService(postgresDb);
  
  await userServicePostgres.createUser(user);
  const retrievedUser2 = await userServicePostgres.getUser(1);
  console.log('Retrieved user:', retrievedUser2);
}

main().catch(console.error);
```

---

## การทดสอบ (Testing)

```typescript
describe('UserService Tests', () => {
  test('should create and get user with in-memory database', async () => {
    const inMemoryDb = new InMemoryDatabase();
    const userService = new UserService(inMemoryDb);

    const user: User = {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com'
    };

    // ทดสอบการสร้าง user
    await expect(userService.createUser(user)).resolves.not.toThrow();

    // ทดสอบการดึง user
    const result = await userService.getUser(1);
    expect(result).toEqual(user);
  });

  test('should create user with MySQL database', async () => {
    const mysqlDb = new MySQLDatabase();
    const userService = new UserService(mysqlDb);

    const user: User = {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com'
    };

    await expect(userService.createUser(user)).resolves.not.toThrow();
  });

  test('should create user with PostgreSQL database', async () => {
    const postgresDb = new PostgreSQLDatabase();
    const userService = new UserService(postgresDb);

    const user: User = {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com'
    };

    await expect(userService.createUser(user)).resolves.not.toThrow();
  });
});
```

---

## สรุป

**DIP ใน TypeScript ช่วยให้:**
- High-level modules ไม่ขึ้นต่อ low-level modules
- ง่ายต่อการทดสอบ
- ง่ายต่อการขยายและแก้ไข
- เพิ่มความยืดหยุ่นของระบบ

**เมื่อไหร่ควรใช้:**
- เมื่อต้องการแยก high-level และ low-level modules
- เมื่อต้องการทดสอบได้ง่าย
- เมื่อต้องการความยืดหยุ่นในการเปลี่ยน implementation

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - การแยกความรับผิดชอบช่วยให้ DIP ทำงานได้ดีขึ้น
- **[Open/Closed Principle](./02-open-closed-principle.md)** - การใช้ interface ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก interface เล็กๆ ช่วยให้ DIP ทำงานได้ดีขึ้น 