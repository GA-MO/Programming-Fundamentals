# Liskov Substitution Principle (LSP) ใน Rust

## ความหมาย
**"ใช้ subtype แทน base type ได้"** - หมายความว่าเราควรสามารถใช้ subtype แทน base type ได้โดยไม่ทำให้โปรแกรมผิดพลาด

## หลักการ
- Subtype ต้องสามารถแทนที่ base type ได้โดยสมบูรณ์
- ต้องรักษา contract ของ base type ไว้
- ต้องไม่ทำให้ precondition แข็งแกร่งขึ้น หรือ postcondition อ่อนแอลง

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```rust
// Logger trait ที่มีปัญหา
trait Logger {
    fn log(&mut self, message: &str) -> Result<(), String>;
    fn get_log_count(&self) -> Result<usize, String>;
}

// FileLogger - บันทึกลงไฟล์
struct FileLogger {
    log_count: usize,
}

impl FileLogger {
    fn new() -> Self {
        FileLogger { log_count: 0 }
    }
}

impl Logger for FileLogger {
    fn log(&mut self, message: &str) -> Result<(), String> {
        println!("Writing to file: {}", message);
        self.log_count += 1;
        Ok(())
    }

    fn get_log_count(&self) -> Result<usize, String> {
        Ok(self.log_count)
    }
}

// DatabaseLogger - บันทึกลงฐานข้อมูล
struct DatabaseLogger;

impl Logger for DatabaseLogger {
    fn log(&mut self, message: &str) -> Result<(), String> {
        println!("Writing to database: {}", message);
        Ok(())
    }

    fn get_log_count(&self) -> Result<usize, String> {
        // ละเมิด LSP! ไม่สามารถนับจำนวน log ได้
        Err("Cannot count logs in database".to_string())
    }
}

// ฟังก์ชันที่ใช้ Logger trait
fn process_logs(logger: &mut dyn Logger, messages: &[&str]) -> Result<(), String> {
    for message in messages {
        logger.log(message)?;
    }
    
    let count = logger.get_log_count()?;
    println!("Total logs written: {}", count);
    // ถ้าเป็น DatabaseLogger จะ error
    Ok(())
}
```

### ปัญหา:
- **DatabaseLogger ไม่สามารถนับ log ได้** แต่ต้อง implement `get_log_count()`
- **ผิด contract** ของ Logger trait
- **ทำให้โปรแกรมผิดพลาด** เมื่อใช้ DatabaseLogger

---

## ✅ ตัวอย่างที่ดี (Good Example)

```rust
// แยก trait ตามความสามารถ
trait Logger {
    fn log(&mut self, message: &str) -> Result<(), String>;
}

trait LogCounter {
    fn get_log_count(&self) -> usize;
}

trait EmailSender {
    fn send(&self, to: &str, subject: &str, message: &str) -> Result<(), String>;
}

// FileLogger - สามารถบันทึกและนับได้
struct FileLogger {
    log_count: usize,
}

impl FileLogger {
    fn new() -> Self {
        FileLogger { log_count: 0 }
    }
}

impl Logger for FileLogger {
    fn log(&mut self, message: &str) -> Result<(), String> {
        println!("Writing to file: {}", message);
        self.log_count += 1;
        Ok(())
    }
}

impl LogCounter for FileLogger {
    fn get_log_count(&self) -> usize {
        self.log_count
    }
}

// DatabaseLogger - สามารถบันทึกได้อย่างเดียว
struct DatabaseLogger;

impl Logger for DatabaseLogger {
    fn log(&mut self, message: &str) -> Result<(), String> {
        println!("Writing to database: {}", message);
        Ok(())
    }
}

// SMTPSender
struct SMTPSender;

impl EmailSender for SMTPSender {
    fn send(&self, to: &str, subject: &str, message: &str) -> Result<(), String> {
        println!("Sending email to {}: {}", to, subject);
        Ok(())
    }
}

// MockSender สำหรับทดสอบ
struct MockSender;

impl EmailSender for MockSender {
    fn send(&self, to: &str, subject: &str, message: &str) -> Result<(), String> {
        println!("Mock: Email sent to {}", to);
        Ok(()) // ไม่ทำอะไรจริง แต่ไม่ error
    }
}

// ฟังก์ชันที่ใช้งาน
fn write_log(logger: &mut dyn Logger, message: &str) -> Result<(), String> {
    logger.log(message)
}

fn count_logs(counter: &dyn LogCounter) {
    let count = counter.get_log_count();
    println!("Total logs: {}", count);
}

fn send_notification(sender: &dyn EmailSender, email: &str, subject: &str, message: &str) -> Result<(), String> {
    sender.send(email, subject, message)
}
```

### ข้อดี:
- **รักษา contract** แต่ละ trait ทำหน้าที่เฉพาะ
- **ไม่มีปัญหา** เมื่อแทนที่กัน
- **เข้าใจง่าย** แยกความรับผิดชอบชัดเจน

---

## ตัวอย่างการใช้งาน

```rust
fn main() -> Result<(), String> {
    let mut file_logger = FileLogger::new();
    let mut db_logger = DatabaseLogger;
    let smtp_sender = SMTPSender;
    let mock_sender = MockSender;

    // ทุก logger สามารถบันทึกได้
    write_log(&mut file_logger, "File log message")?;
    write_log(&mut db_logger, "Database log message")?;

    // เฉพาะ FileLogger เท่านั้นที่นับได้
    count_logs(&file_logger);
    // count_logs(&db_logger); // Compile error - ดีแล้ว!

    // ทุก sender สามารถส่งอีเมลได้
    send_notification(&smtp_sender, "user@example.com", "Welcome", "Hello!")?;
    send_notification(&mock_sender, "test@example.com", "Test", "Test message")?;

    Ok(())
}
```

---

## การทดสอบ

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_loggers_should_log_correctly() {
        let mut file_logger = FileLogger::new();
        let mut db_logger = DatabaseLogger;

        // ทั้งสองตัวควรบันทึกได้
        assert!(file_logger.log("test message").is_ok());
        assert!(db_logger.log("test message").is_ok());

        // เฉพาะ FileLogger เท่านั้นที่นับได้
        assert_eq!(1, file_logger.get_log_count());
    }

    #[test]
    fn test_email_senders_are_substitutable() {
        let smtp_sender = SMTPSender;
        let mock_sender = MockSender;

        // ทั้งสองตัวควรส่งได้โดยไม่ error
        assert!(smtp_sender.send("test@example.com", "Test", "Message").is_ok());
        assert!(mock_sender.send("test@example.com", "Test", "Message").is_ok());
    }
}
```

---

## ตัวอย่างเพิ่มเติม: การบันทึกข้อมูล

```rust
// DataSaver trait
trait DataSaver {
    fn save(&self, data: &str) -> Result<(), String>;
}

// FileSaver
struct FileSaver;

impl DataSaver for FileSaver {
    fn save(&self, data: &str) -> Result<(), String> {
        println!("Saving to file: {}", data);
        Ok(())
    }
}

// DatabaseSaver
struct DatabaseSaver;

impl DataSaver for DatabaseSaver {
    fn save(&self, data: &str) -> Result<(), String> {
        println!("Saving to database: {}", data);
        Ok(())
    }
}

// MockSaver
struct MockSaver;

impl DataSaver for MockSaver {
    fn save(&self, data: &str) -> Result<(), String> {
        println!("Mock save: {}", data);
        Ok(()) // สำหรับทดสอบ
    }
}

fn save_user_data(saver: &dyn DataSaver, user_data: &str) -> Result<(), String> {
    saver.save(user_data)
}
```

---

## สรุป

**LSP ใน Rust ช่วยให้:**
- โค้ดทำงานได้ถูกต้องเมื่อแทนที่ type
- ลดข้อผิดพลาดในการใช้งาน
- ง่ายต่อการทดสอบ
- เพิ่มความยืดหยุ่นของโค้ด

**หลักการสำคัญ:**
- แยก trait ตามความสามารถ
- รักษา contract ของ trait
- ไม่ทำให้เกิดข้อผิดพลาดที่ไม่คาดคิด

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก trait เล็กๆ ช่วยให้ LSP ทำงานได้ดีขึ้น
- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ trait ช่วยให้ LSP ทำงานได้ดีขึ้น 