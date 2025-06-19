# Single Responsibility Principle (SRP) ใน Rust

## ความหมาย
**"หนึ่ง struct/trait ควรมีหน้าที่เดียว"** - หมายความว่าแต่ละ struct หรือ function ควรมีเหตุผลเดียวที่จะเปลี่ยนแปลง

## หลักการ
- หนึ่ง struct = หนึ่งหน้าที่
- ถ้ามีเหตุผลหลายอย่างที่จะเปลี่ยนแปลง struct แสดงว่าผิดหลักการนี้
- ช่วยให้โค้ดอ่านง่าย ทดสอบง่าย และดูแลรักษาง่าย

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```rust
// UserService ที่ทำหลายหน้าที่เกินไป
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct User {
    name: String,
    email: String,
    password: String,
}

#[derive(Debug)]
struct UserService {
    db_connection: String,
}

impl UserService {
    fn new() -> Self {
        UserService {
            db_connection: "mysql://localhost:3306/users".to_string(),
        }
    }

    fn create_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        // 1. ตรวจสอบข้อมูล
        if user.name.is_empty() || user.email.is_empty() {
            return Err("Name and email are required".into());
        }

        // 2. เข้ารหัส password
        let hashed_password = self.hash_password(&user.password)?;

        // 3. บันทึกลงฐานข้อมูล
        self.save_to_database(&user.name, &user.email, &hashed_password)?;

        // 4. ส่งอีเมลยืนยัน
        self.send_welcome_email(&user.email, &user.name)?;

        // 5. บันทึก log
        println!("User created: {}", user.email);

        Ok(())
    }

    fn hash_password(&self, password: &str) -> Result<String, Box<dyn Error>> {
        // โค้ดเข้ารหัส password
        Ok(format!("hashed_{}", password))
    }

    fn save_to_database(&self, name: &str, email: &str, password: &str) -> Result<(), Box<dyn Error>> {
        println!("Saving to database: {} - {}", name, email);
        Ok(())
    }

    fn send_welcome_email(&self, email: &str, name: &str) -> Result<(), Box<dyn Error>> {
        println!("Sending welcome email to {}: {}", email, name);
        Ok(())
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

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug, Clone)]
struct User {
    name: String,
    email: String,
    password: String,
}

// 1. UserValidator - ตรวจสอบข้อมูล
struct UserValidator;

impl UserValidator {
    fn validate(user: &User) -> Result<(), Box<dyn Error>> {
        if user.name.is_empty() || user.email.is_empty() {
            return Err("Name and email are required".into());
        }
        Ok(())
    }
}

// 2. PasswordHasher - เข้ารหัส password
struct PasswordHasher;

impl PasswordHasher {
    fn hash(password: &str) -> Result<String, Box<dyn Error>> {
        Ok(format!("hashed_{}", password))
    }
}

// 3. UserRepository - จัดการฐานข้อมูล
struct UserRepository {
    connection: String,
}

impl UserRepository {
    fn new() -> Self {
        UserRepository {
            connection: "mysql://localhost:3306/users".to_string(),
        }
    }

    fn create(&self, user: &User) -> Result<(), Box<dyn Error>> {
        println!("Saving to database: {} - {}", user.name, user.email);
        Ok(())
    }
}

// 4. EmailService - ส่งอีเมล
struct EmailService;

impl EmailService {
    fn send_welcome_email(user: &User) -> Result<(), Box<dyn Error>> {
        println!("Sending welcome email to {}: {}", user.email, user.name);
        Ok(())
    }
}

// 5. Logger - บันทึก log
struct Logger;

impl Logger {
    fn log_user_created(email: &str) {
        println!("User created: {}", email);
    }
}

// 6. UserService - ประสานงานระหว่างส่วนต่างๆ
struct UserService {
    repository: UserRepository,
}

impl UserService {
    fn new() -> Self {
        UserService {
            repository: UserRepository::new(),
        }
    }

    fn create_user(&self, user: &User) -> Result<(), Box<dyn Error>> {
        // 1. ตรวจสอบข้อมูล
        UserValidator::validate(user)?;

        // 2. เข้ารหัส password
        let hashed_password = PasswordHasher::hash(&user.password)?;
        let mut user_with_hash = user.clone();
        user_with_hash.password = hashed_password;

        // 3. บันทึกลงฐานข้อมูล
        self.repository.create(&user_with_hash)?;

        // 4. ส่งอีเมลยืนยัน
        EmailService::send_welcome_email(user)?;

        // 5. บันทึก log
        Logger::log_user_created(&user.email);

        Ok(())
    }
}
```

### ข้อดีของโค้ดที่ดี:
1. **แต่ละ struct มีหน้าที่เดียว**: ง่ายต่อการเข้าใจและแก้ไข
2. **ทดสอบง่าย**: สามารถทดสอบแต่ละส่วนแยกกันได้
3. **นำกลับใช้ได้**: สามารถใช้ PasswordHasher ในที่อื่นได้
4. **ขยายง่าย**: ถ้าต้องการเปลี่ยนวิธีส่งอีเมล แค่แก้ไข EmailService
5. **Dependency Injection**: ง่ายต่อการ mock และทดสอบ

---

## ตัวอย่างการใช้งานจริง

```rust
fn main() -> Result<(), Box<dyn Error>> {
    let user_service = UserService::new();

    let user = User {
        name: "John Doe".to_string(),
        email: "john@example.com".to_string(),
        password: "password123".to_string(),
    };

    user_service.create_user(&user)?;
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
    fn test_user_validator_valid_user() {
        let user = User {
            name: "John".to_string(),
            email: "john@example.com".to_string(),
            password: "password".to_string(),
        };

        let result = UserValidator::validate(&user);
        assert!(result.is_ok());
    }

    #[test]
    fn test_user_validator_invalid_user() {
        let user = User {
            name: "".to_string(),
            email: "".to_string(),
            password: "password".to_string(),
        };

        let result = UserValidator::validate(&user);
        assert!(result.is_err());
    }

    #[test]
    fn test_password_hasher() {
        let password = "password123";
        let result = PasswordHasher::hash(password);
        
        assert!(result.is_ok());
        let hashed = result.unwrap();
        assert!(hashed.starts_with("hashed_"));
    }

    #[test]
    fn test_user_service_create_user() {
        let user_service = UserService::new();
        let user = User {
            name: "John".to_string(),
            email: "john@example.com".to_string(),
            password: "password123".to_string(),
        };

        let result = user_service.create_user(&user);
        assert!(result.is_ok());
    }
}
```

---

## สรุป

**SRP ใน Rust ช่วยให้:**
- โค้ดอ่านง่ายและเข้าใจง่าย
- ทดสอบได้ง่าย
- แก้ไขและขยายได้ง่าย
- ลดการพึ่งพากันระหว่างส่วนต่างๆ
- เพิ่มความสามารถในการนำกลับใช้

**เมื่อไหร่ควรใช้:**
- เมื่อ struct หรือ function มีหน้าที่หลายอย่าง
- เมื่อยากต่อการทดสอบ
- เมื่อแก้ไขส่วนหนึ่งกระทบส่วนอื่น

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ trait ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก trait เล็กๆ ช่วยให้ SRP ชัดเจนขึ้น 