# Liskov Substitution Principle (LSP) ใน TypeScript

## ความหมาย
**"ใช้ subtype แทน base type ได้"** - หมายความว่าเราควรสามารถใช้ subtype แทน base type ได้โดยไม่ทำให้โปรแกรมผิดพลาด

## หลักการ
- Subtype ต้องสามารถแทนที่ base type ได้โดยสมบูรณ์
- ต้องรักษา contract ของ base type ไว้
- ต้องไม่ทำให้ precondition แข็งแกร่งขึ้น หรือ postcondition อ่อนแอลง

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```typescript
// Logger interface ที่มีปัญหา
interface Logger {
  log(message: string): Promise<void>;
  getLogCount(): Promise<number>;
}

// FileLogger - บันทึกลงไฟล์
class FileLogger implements Logger {
  private logCount = 0;

  async log(message: string): Promise<void> {
    console.log(`Writing to file: ${message}`);
    this.logCount++;
  }

  async getLogCount(): Promise<number> {
    return this.logCount;
  }
}

// DatabaseLogger - บันทึกลงฐานข้อมูล
class DatabaseLogger implements Logger {
  async log(message: string): Promise<void> {
    console.log(`Writing to database: ${message}`);
  }

  async getLogCount(): Promise<number> {
    // ละเมิด LSP! ไม่สามารถนับจำนวน log ได้
    throw new Error('Cannot count logs in database');
  }
}

// ฟังก์ชันที่ใช้ Logger interface
async function processLogs(logger: Logger, messages: string[]): Promise<void> {
  for (const message of messages) {
    await logger.log(message);
  }
  
  const count = await logger.getLogCount();
  console.log(`Total logs written: ${count}`);
  // ถ้าเป็น DatabaseLogger จะ error
}
```

### ปัญหา:
- **DatabaseLogger ไม่สามารถนับ log ได้** แต่ต้อง implement `getLogCount()`
- **ผิด contract** ของ Logger interface
- **ทำให้โปรแกรมผิดพลาด** เมื่อใช้ DatabaseLogger

---

## ✅ ตัวอย่างที่ดี (Good Example)

```typescript
// แยก interface ตามความสามารถ
interface Logger {
  log(message: string): Promise<void>;
}

interface LogCounter {
  getLogCount(): number;
}

interface EmailSender {
  send(to: string, subject: string, message: string): Promise<void>;
}

// FileLogger - สามารถบันทึกและนับได้
class FileLogger implements Logger, LogCounter {
  private logCount = 0;

  async log(message: string): Promise<void> {
    console.log(`Writing to file: ${message}`);
    this.logCount++;
  }

  getLogCount(): number {
    return this.logCount;
  }
}

// DatabaseLogger - สามารถบันทึกได้อย่างเดียว
class DatabaseLogger implements Logger {
  async log(message: string): Promise<void> {
    console.log(`Writing to database: ${message}`);
  }
}

// SMTPSender
class SMTPSender implements EmailSender {
  async send(to: string, subject: string, message: string): Promise<void> {
    console.log(`Sending email to ${to}: ${subject}`);
  }
}

// MockSender สำหรับทดสอบ
class MockSender implements EmailSender {
  async send(to: string, subject: string, message: string): Promise<void> {
    console.log(`Mock: Email sent to ${to}`);
    // ไม่ทำอะไรจริง แต่ไม่ error
  }
}

// ฟังก์ชันที่ใช้งาน
async function writeLog(logger: Logger, message: string): Promise<void> {
  await logger.log(message);
}

function countLogs(counter: LogCounter): void {
  const count = counter.getLogCount();
  console.log(`Total logs: ${count}`);
}

async function sendNotification(sender: EmailSender, email: string, subject: string, message: string): Promise<void> {
  await sender.send(email, subject, message);
}
```

### ข้อดี:
- **รักษา contract** แต่ละ interface ทำหน้าที่เฉพาะ
- **ไม่มีปัญหา** เมื่อแทนที่กัน
- **เข้าใจง่าย** แยกความรับผิดชอบชัดเจน

---

## ตัวอย่างการใช้งาน

```typescript
async function main(): Promise<void> {
  const fileLogger = new FileLogger();
  const dbLogger = new DatabaseLogger();
  const smtpSender = new SMTPSender();
  const mockSender = new MockSender();

  // ทุก logger สามารถบันทึกได้
  await writeLog(fileLogger, 'File log message');
  await writeLog(dbLogger, 'Database log message');

  // เฉพาะ FileLogger เท่านั้นที่นับได้
  countLogs(fileLogger);
  // countLogs(dbLogger); // Compile error - ดีแล้ว!

  // ทุก sender สามารถส่งอีเมลได้
  await sendNotification(smtpSender, 'user@example.com', 'Welcome', 'Hello!');
  await sendNotification(mockSender, 'test@example.com', 'Test', 'Test message');
}

main().catch(console.error);
```

---

## การทดสอบ

```typescript
describe('Logger Tests', () => {
  test('Loggers should log correctly', async () => {
    const fileLogger = new FileLogger();
    const dbLogger = new DatabaseLogger();

    // ทั้งสองตัวควรบันทึกได้
    await expect(fileLogger.log('test message')).resolves.not.toThrow();
    await expect(dbLogger.log('test message')).resolves.not.toThrow();

    // เฉพาะ FileLogger เท่านั้นที่นับได้
    expect(fileLogger.getLogCount()).toBe(1);
  });

  test('Email senders are substitutable', async () => {
    const smtpSender = new SMTPSender();
    const mockSender = new MockSender();

    // ทั้งสองตัวควรส่งได้โดยไม่ error
    await expect(smtpSender.send('test@example.com', 'Test', 'Message')).resolves.not.toThrow();
    await expect(mockSender.send('test@example.com', 'Test', 'Message')).resolves.not.toThrow();
  });
});
```

---

## ตัวอย่างเพิ่มเติม: การบันทึกข้อมูล

```typescript
// DataSaver interface
interface DataSaver {
  save(data: string): Promise<void>;
}

// FileSaver
class FileSaver implements DataSaver {
  async save(data: string): Promise<void> {
    console.log(`Saving to file: ${data}`);
  }
}

// DatabaseSaver
class DatabaseSaver implements DataSaver {
  async save(data: string): Promise<void> {
    console.log(`Saving to database: ${data}`);
  }
}

// MockSaver
class MockSaver implements DataSaver {
  async save(data: string): Promise<void> {
    console.log(`Mock save: ${data}`);
    // สำหรับทดสอบ
  }
}

async function saveUserData(saver: DataSaver, userData: string): Promise<void> {
  await saver.save(userData);
}
```

---

## สรุป

**LSP ใน TypeScript ช่วยให้:**
- โค้ดทำงานได้ถูกต้องเมื่อแทนที่ type
- ลดข้อผิดพลาดในการใช้งาน
- ง่ายต่อการทดสอบ
- เพิ่มความยืดหยุ่นของโค้ด

**หลักการสำคัญ:**
- แยก interface ตามความสามารถ
- รักษา contract ของ interface
- ไม่ทำให้เกิดข้อผิดพลาดที่ไม่คาดคิด

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Interface Segregation Principle](./04-interface-segregation-principle.md)** - การแยก interface เล็กๆ ช่วยให้ LSP ทำงานได้ดีขึ้น
- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface ช่วยให้ LSP ทำงานได้ดีขึ้น 