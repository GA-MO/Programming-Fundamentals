# Interface Segregation Principle (ISP) ใน Rust

## ความหมาย
**"แยก trait ให้เล็กลง"** - หมายความว่า client ไม่ควรถูกบังคับให้ implement method ที่ไม่ได้ใช้

## หลักการ
- แยก trait ใหญ่เป็น trait เล็กๆ หลายตัว
- Client ควร implement เฉพาะ trait ที่จำเป็น
- หลีกเลี่ยง "fat interface" ที่มี method มากเกินไป

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```rust
// Fat trait ที่มี method มากเกินไป
trait Worker {
    fn work(&self) -> Result<(), String>;
    fn eat(&self) -> Result<(), String>;
    fn sleep(&self) -> Result<(), String>;
    fn get_salary(&self) -> f64;
    fn take_vacation(&self, days: u32) -> Result<(), String>;
    fn submit_report(&self) -> Result<(), String>;
    fn attend_meeting(&self) -> Result<(), String>;
}

// Human Worker ใช้ได้ทุก method
struct HumanWorker {
    name: String,
    salary: f64,
}

impl Worker for HumanWorker {
    fn work(&self) -> Result<(), String> {
        println!("{} is working", self.name);
        Ok(())
    }

    fn eat(&self) -> Result<(), String> {
        println!("{} is eating", self.name);
        Ok(())
    }

    fn sleep(&self) -> Result<(), String> {
        println!("{} is sleeping", self.name);
        Ok(())
    }

    fn get_salary(&self) -> f64 {
        self.salary
    }

    fn take_vacation(&self, days: u32) -> Result<(), String> {
        println!("{} is taking {} days vacation", self.name, days);
        Ok(())
    }

    fn submit_report(&self) -> Result<(), String> {
        println!("{} is submitting report", self.name);
        Ok(())
    }

    fn attend_meeting(&self) -> Result<(), String> {
        println!("{} is attending meeting", self.name);
        Ok(())
    }
}

// Robot Worker ไม่สามารถใช้บาง method ได้
struct RobotWorker {
    id: String,
    power_level: u32,
}

impl Worker for RobotWorker {
    fn work(&self) -> Result<(), String> {
        println!("Robot {} is working", self.id);
        Ok(())
    }

    fn eat(&self) -> Result<(), String> {
        // Robot ไม่กินอาหาร แต่ต้อง implement
        Err("Robot cannot eat".to_string())
    }

    fn sleep(&self) -> Result<(), String> {
        // Robot ไม่นอน แต่ต้อง implement
        Err("Robot cannot sleep".to_string())
    }

    fn get_salary(&self) -> f64 {
        // Robot ไม่ได้รับเงินเดือน
        0.0
    }

    fn take_vacation(&self, _days: u32) -> Result<(), String> {
        // Robot ไม่ลาพักร้อน
        Err("Robot cannot take vacation".to_string())
    }

    fn submit_report(&self) -> Result<(), String> {
        println!("Robot {} is submitting report", self.id);
        Ok(())
    }

    fn attend_meeting(&self) -> Result<(), String> {
        println!("Robot {} is attending meeting", self.id);
        Ok(())
    }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **Robot ถูกบังคับให้ implement method ที่ไม่ได้ใช้**: เช่น eat, sleep, take_vacation
2. **Fat interface**: Worker trait มี method มากเกินไป
3. **ยากต่อการขยาย**: ถ้าเพิ่ม method ใหม่ ทุก implementer ต้อง implement
4. **ผิดหลัก LSP**: Robot ไม่สามารถแทนที่ Worker ได้อย่างสมบูรณ์

---

## ✅ ตัวอย่างที่ดี (Good Example)

```rust
// แยก trait ตามความสามารถ
trait Workable {
    fn work(&self) -> Result<(), String>;
}

trait Eatable {
    fn eat(&self) -> Result<(), String>;
}

trait Sleepable {
    fn sleep(&self) -> Result<(), String>;
}

trait Payable {
    fn get_salary(&self) -> f64;
}

trait Vacationable {
    fn take_vacation(&self, days: u32) -> Result<(), String>;
}

trait Reportable {
    fn submit_report(&self) -> Result<(), String>;
}

trait MeetingAttendable {
    fn attend_meeting(&self) -> Result<(), String>;
}

// Human Worker สามารถทำได้ทุกอย่าง
struct HumanWorker {
    name: String,
    salary: f64,
}

impl Workable for HumanWorker {
    fn work(&self) -> Result<(), String> {
        println!("{} is working", self.name);
        Ok(())
    }
}

impl Eatable for HumanWorker {
    fn eat(&self) -> Result<(), String> {
        println!("{} is eating", self.name);
        Ok(())
    }
}

impl Sleepable for HumanWorker {
    fn sleep(&self) -> Result<(), String> {
        println!("{} is sleeping", self.name);
        Ok(())
    }
}

impl Payable for HumanWorker {
    fn get_salary(&self) -> f64 {
        self.salary
    }
}

impl Vacationable for HumanWorker {
    fn take_vacation(&self, days: u32) -> Result<(), String> {
        println!("{} is taking {} days vacation", self.name, days);
        Ok(())
    }
}

impl Reportable for HumanWorker {
    fn submit_report(&self) -> Result<(), String> {
        println!("{} is submitting report", self.name);
        Ok(())
    }
}

impl MeetingAttendable for HumanWorker {
    fn attend_meeting(&self) -> Result<(), String> {
        println!("{} is attending meeting", self.name);
        Ok(())
    }
}

// Robot Worker เฉพาะที่จำเป็น
struct RobotWorker {
    id: String,
    power_level: u32,
}

impl Workable for RobotWorker {
    fn work(&self) -> Result<(), String> {
        println!("Robot {} is working", self.id);
        Ok(())
    }
}

impl Reportable for RobotWorker {
    fn submit_report(&self) -> Result<(), String> {
        println!("Robot {} is submitting report", self.id);
        Ok(())
    }
}

impl MeetingAttendable for RobotWorker {
    fn attend_meeting(&self) -> Result<(), String> {
        println!("Robot {} is attending meeting", self.id);
        Ok(())
    }
}

// Functions ที่ใช้ trait เฉพาะ
fn make_work(worker: &dyn Workable) -> Result<(), String> {
    worker.work()
}

fn make_eat(worker: &dyn Eatable) -> Result<(), String> {
    worker.eat()
}

fn make_sleep(worker: &dyn Sleepable) -> Result<(), String> {
    worker.sleep()
}

fn get_worker_salary(worker: &dyn Payable) -> f64 {
    worker.get_salary()
}

fn make_take_vacation(worker: &dyn Vacationable, days: u32) -> Result<(), String> {
    worker.take_vacation(days)
}

fn make_submit_report(worker: &dyn Reportable) -> Result<(), String> {
    worker.submit_report()
}

fn make_attend_meeting(worker: &dyn MeetingAttendable) -> Result<(), String> {
    worker.attend_meeting()
}
```

### ข้อดีของโค้ดที่ดี:
1. **Client implement เฉพาะที่จำเป็น**: Robot ไม่ต้อง implement method ที่ไม่ได้ใช้
2. **แยกความรับผิดชอบ**: แต่ละ trait มีหน้าที่เดียว
3. **ขยายง่าย**: เพิ่ม trait ใหม่ได้โดยไม่กระทบส่วนอื่น
4. **ยืดหยุ่น**: สามารถผสมผสาน trait ได้ตามต้องการ

---

## ตัวอย่างการใช้งานจริง

```rust
fn main() -> Result<(), String> {
    let human = HumanWorker {
        name: "John".to_string(),
        salary: 50000.0,
    };

    let robot = RobotWorker {
        id: "R2D2".to_string(),
        power_level: 100,
    };

    // Human สามารถทำได้ทุกอย่าง
    make_work(&human)?;
    make_eat(&human)?;
    make_sleep(&human)?;
    println!("Human salary: ${}", get_worker_salary(&human));
    make_take_vacation(&human, 5)?;
    make_submit_report(&human)?;
    make_attend_meeting(&human)?;

    // Robot เฉพาะที่จำเป็น
    make_work(&robot)?;
    make_submit_report(&robot)?;
    make_attend_meeting(&robot)?;
    // make_eat(&robot); // จะ compile error เพราะ Robot ไม่ implement Eatable

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
    fn test_human_worker_all_capabilities() {
        let human = HumanWorker {
            name: "John".to_string(),
            salary: 50000.0,
        };

        assert!(make_work(&human).is_ok());
        assert!(make_eat(&human).is_ok());
        assert!(make_sleep(&human).is_ok());
        assert_eq!(get_worker_salary(&human), 50000.0);
        assert!(make_take_vacation(&human, 5).is_ok());
        assert!(make_submit_report(&human).is_ok());
        assert!(make_attend_meeting(&human).is_ok());
    }

    #[test]
    fn test_robot_worker_limited_capabilities() {
        let robot = RobotWorker {
            id: "R2D2".to_string(),
            power_level: 100,
        };

        assert!(make_work(&robot).is_ok());
        assert!(make_submit_report(&robot).is_ok());
        assert!(make_attend_meeting(&robot).is_ok());
        // robot ไม่สามารถ eat, sleep, take_vacation ได้
    }

    #[test]
    fn test_workable_trait() {
        let human = HumanWorker {
            name: "John".to_string(),
            salary: 50000.0,
        };
        let robot = RobotWorker {
            id: "R2D2".to_string(),
            power_level: 100,
        };

        assert!(make_work(&human).is_ok());
        assert!(make_work(&robot).is_ok());
    }
}
```

---

## สรุป

**ISP ใน Rust ช่วยให้:**
- Client implement เฉพาะ trait ที่จำเป็น
- แยกความรับผิดชอบได้ชัดเจน
- ง่ายต่อการขยายและดูแลรักษา
- ลดการพึ่งพากันระหว่างส่วนต่างๆ

**เมื่อไหร่ควรใช้:**
- เมื่อมี fat trait ที่มี method มากเกินไป
- เมื่อ client ถูกบังคับให้ implement method ที่ไม่ได้ใช้
- เมื่อต้องการความยืดหยุ่นในการผสมผสาน trait

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - การแยก trait เล็กๆ ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Liskov Substitution Principle](./03-liskov-substitution-principle.md)** - การแยก trait ช่วยให้ LSP ทำงานได้ดีขึ้น 