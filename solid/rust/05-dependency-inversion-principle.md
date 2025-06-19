# Dependency Inversion Principle (DIP) ใน Rust

## ความหมาย
**"ขึ้นต่อ abstraction ไม่ใช่ concrete"** - หมายความว่า high-level modules ไม่ควรขึ้นต่อ low-level modules แต่ควรขึ้นต่อ abstraction

## หลักการ
- High-level modules ไม่ควรขึ้นต่อ low-level modules
- ทั้งสองควรขึ้นต่อ abstraction
- Abstraction ไม่ควรขึ้นต่อ details
- Details ควรขึ้นต่อ abstraction

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```rust
// High-level module ขึ้นต่อ low-level module โดยตรง
use std::error::Error;

// Low-level module
struct MySQLDatabase {
    connection_string: String,
}

impl MySQLDatabase {
    fn new() -> Self {
        MySQLDatabase {
            connection_string: "mysql://localhost:3306/users".to_string(),
        }
    }

    fn save_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        println!("Saving user to MySQL: {}", user.name);
        Ok(())
    }

    fn get_user(&self, id: u32) -> Result<User, Box<dyn Error>> {
        println!("Getting user from MySQL: {}", id);
        Ok(User {
            id,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
        })
    }
}

// High-level module ขึ้นต่อ low-level module โดยตรง
struct UserService {
    database: MySQLDatabase,
}

impl UserService {
    fn new() -> Self {
        UserService {
            database: MySQLDatabase::new(),
        }
    }

    fn create_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        self.database.save_user(user)
    }

    fn get_user(&self, id: u32) -> Result<User, Box<dyn Error>> {
        self.database.get_user(id)
    }
}

#[derive(Debug)]
struct User {
    id: u32,
    name: String,
    email: String,
}
```

### ปัญหาของโค้ดข้างต้น:
1. **UserService ขึ้นต่อ MySQLDatabase โดยตรง**: ถ้าต้องการเปลี่ยนเป็น PostgreSQL ต้องแก้ไข UserService
2. **ยากต่อการทดสอบ**: ต้องใช้ MySQL จริงในการทดสอบ
3. **ยากต่อการขยาย**: ถ้าต้องการเพิ่ม database ใหม่ต้องแก้ไข UserService
4. **ผิดหลัก DIP**: High-level module ขึ้นต่อ low-level module

---

## ✅ ตัวอย่างที่ดี (Good Example)

```rust
use std::error::Error;

#[derive(Debug)]
struct User {
    id: u32,
    name: String,
    email: String,
}

// Abstraction (trait)
trait UserRepository {
    fn save_user(&self, user: &User) -> Result<(), Box<dyn Error>>;
    fn get_user(&self, id: u32) -> Result<User, Box<dyn Error>>;
}

// Low-level module - MySQL implementation
struct MySQLDatabase {
    connection_string: String,
}

impl MySQLDatabase {
    fn new() -> Self {
        MySQLDatabase {
            connection_string: "mysql://localhost:3306/users".to_string(),
        }
    }
}

impl UserRepository for MySQLDatabase {
    fn save_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        println!("Saving user to MySQL: {}", user.name);
        Ok(())
    }

    fn get_user(&self, id: u32) -> Result<User, Box<dyn Error>> {
        println!("Getting user from MySQL: {}", id);
        Ok(User {
            id,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
        })
    }
}

// Low-level module - PostgreSQL implementation
struct PostgreSQLDatabase {
    connection_string: String,
}

impl PostgreSQLDatabase {
    fn new() -> Self {
        PostgreSQLDatabase {
            connection_string: "postgresql://localhost:5432/users".to_string(),
        }
    }
}

impl UserRepository for PostgreSQLDatabase {
    fn save_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        println!("Saving user to PostgreSQL: {}", user.name);
        Ok(())
    }

    fn get_user(&self, id: u32) -> Result<User, Box<dyn Error>> {
        println!("Getting user from PostgreSQL: {}", id);
        Ok(User {
            id,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
        })
    }
}

// Low-level module - In-memory implementation (for testing)
struct InMemoryDatabase {
    users: std::collections::HashMap<u32, User>,
}

impl InMemoryDatabase {
    fn new() -> Self {
        InMemoryDatabase {
            users: std::collections::HashMap::new(),
        }
    }
}

impl UserRepository for InMemoryDatabase {
    fn save_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        println!("Saving user to InMemory: {}", user.name);
        // ในความเป็นจริงจะต้องใช้ Arc<Mutex<>> สำหรับ thread safety
        Ok(())
    }

    fn get_user(&self, id: u32) -> Result<User, Box<dyn Error>> {
        println!("Getting user from InMemory: {}", id);
        self.users.get(&id)
            .cloned()
            .ok_or_else(|| "User not found".into())
    }
}

// High-level module ขึ้นต่อ abstraction
struct UserService {
    repository: Box<dyn UserRepository>,
}

impl UserService {
    fn new(repository: Box<dyn UserRepository>) -> Self {
        UserService { repository }
    }

    fn create_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        self.repository.save_user(user)
    }

    fn get_user(&self, id: u32) -> Result<User, Box<dyn Error>> {
        self.repository.get_user(id)
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

```rust
fn main() -> Result<(), Box<dyn Error>> {
    // ใช้ MySQL
    let mysql_db = MySQLDatabase::new();
    let user_service_mysql = UserService::new(Box::new(mysql_db));
    
    let user = User {
        id: 1,
        name: "John Doe".to_string(),
        email: "john@example.com".to_string(),
    };
    
    user_service_mysql.create_user(&user)?;
    let retrieved_user = user_service_mysql.get_user(1)?;
    println!("Retrieved user: {:?}", retrieved_user);

    // เปลี่ยนเป็น PostgreSQL
    let postgres_db = PostgreSQLDatabase::new();
    let user_service_postgres = UserService::new(Box::new(postgres_db));
    
    user_service_postgres.create_user(&user)?;
    let retrieved_user = user_service_postgres.get_user(1)?;
    println!("Retrieved user: {:?}", retrieved_user);

    Ok(())
}
```

---

## การทดสอบ (Testing)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_user_service_with_in_memory_database() {
        let in_memory_db = InMemoryDatabase::new();
        let user_service = UserService::new(Box::new(in_memory_db));

        let user = User {
            id: 1,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
        };

        // ทดสอบการสร้าง user
        let result = user_service.create_user(&user);
        assert!(result.is_ok());

        // ทดสอบการดึง user
        let result = user_service.get_user(1);
        assert!(result.is_ok());
    }

    #[test]
    fn test_user_service_with_mysql_database() {
        let mysql_db = MySQLDatabase::new();
        let user_service = UserService::new(Box::new(mysql_db));

        let user = User {
            id: 1,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
        };

        let result = user_service.create_user(&user);
        assert!(result.is_ok());
    }

    #[test]
    fn test_user_service_with_postgresql_database() {
        let postgres_db = PostgreSQLDatabase::new();
        let user_service = UserService::new(Box::new(postgres_db));

        let user = User {
            id: 1,
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
        };

        let result = user_service.create_user(&user);
        assert!(result.is_ok());
    }
}
```

---

## สรุป

**DIP ใน Rust ช่วยให้:**
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
- **[Open/Closed Principle](./02-open-closed-principle.md)** - การใช้ trait ช่วยให้ OCP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก trait เล็กๆ ช่วยให้ DIP ทำงานได้ดีขึ้น 