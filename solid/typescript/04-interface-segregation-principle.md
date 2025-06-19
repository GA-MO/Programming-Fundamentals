# Interface Segregation Principle (ISP) ใน TypeScript

## ความหมาย
**"แยก interface ให้เล็กลง"** - หมายความว่า client ไม่ควรถูกบังคับให้ implement method ที่ไม่ได้ใช้

## หลักการ
- แยก interface ใหญ่เป็น interface เล็กๆ หลายตัว
- Client ควร implement เฉพาะ interface ที่จำเป็น
- หลีกเลี่ยง "fat interface" ที่มี method มากเกินไป

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

```typescript
// Fat interface ที่มี method มากเกินไป
interface Worker {
  work(): Promise<void>;
  eat(): Promise<void>;
  sleep(): Promise<void>;
  getSalary(): number;
  takeVacation(days: number): Promise<void>;
  submitReport(): Promise<void>;
  attendMeeting(): Promise<void>;
}

// Human Worker ใช้ได้ทุก method
class HumanWorker implements Worker {
  constructor(
    private name: string,
    private salary: number
  ) {}

  async work(): Promise<void> {
    console.log(`${this.name} is working`);
  }

  async eat(): Promise<void> {
    console.log(`${this.name} is eating`);
  }

  async sleep(): Promise<void> {
    console.log(`${this.name} is sleeping`);
  }

  getSalary(): number {
    return this.salary;
  }

  async takeVacation(days: number): Promise<void> {
    console.log(`${this.name} is taking ${days} days vacation`);
  }

  async submitReport(): Promise<void> {
    console.log(`${this.name} is submitting report`);
  }

  async attendMeeting(): Promise<void> {
    console.log(`${this.name} is attending meeting`);
  }
}

// Robot Worker ไม่สามารถใช้บาง method ได้
class RobotWorker implements Worker {
  constructor(
    private id: string,
    private powerLevel: number
  ) {}

  async work(): Promise<void> {
    console.log(`Robot ${this.id} is working`);
  }

  async eat(): Promise<void> {
    // Robot ไม่กินอาหาร แต่ต้อง implement
    throw new Error('Robot cannot eat');
  }

  async sleep(): Promise<void> {
    // Robot ไม่นอน แต่ต้อง implement
    throw new Error('Robot cannot sleep');
  }

  getSalary(): number {
    // Robot ไม่ได้รับเงินเดือน
    return 0;
  }

  async takeVacation(days: number): Promise<void> {
    // Robot ไม่ลาพักร้อน
    throw new Error('Robot cannot take vacation');
  }

  async submitReport(): Promise<void> {
    console.log(`Robot ${this.id} is submitting report`);
  }

  async attendMeeting(): Promise<void> {
    console.log(`Robot ${this.id} is attending meeting`);
  }
}
```

### ปัญหาของโค้ดข้างต้น:
1. **Robot ถูกบังคับให้ implement method ที่ไม่ได้ใช้**: เช่น eat, sleep, takeVacation
2. **Fat interface**: Worker interface มี method มากเกินไป
3. **ยากต่อการขยาย**: ถ้าเพิ่ม method ใหม่ ทุก implementer ต้อง implement
4. **ผิดหลัก LSP**: Robot ไม่สามารถแทนที่ Worker ได้อย่างสมบูรณ์

---

## ✅ ตัวอย่างที่ดี (Good Example)

```typescript
// แยก interface ตามความสามารถ
interface Workable {
  work(): Promise<void>;
}

interface Eatable {
  eat(): Promise<void>;
}

interface Sleepable {
  sleep(): Promise<void>;
}

interface Payable {
  getSalary(): number;
}

interface Vacationable {
  takeVacation(days: number): Promise<void>;
}

interface Reportable {
  submitReport(): Promise<void>;
}

interface MeetingAttendable {
  attendMeeting(): Promise<void>;
}

// Human Worker สามารถทำได้ทุกอย่าง
class HumanWorker implements Workable, Eatable, Sleepable, Payable, Vacationable, Reportable, MeetingAttendable {
  constructor(
    private name: string,
    private salary: number
  ) {}

  async work(): Promise<void> {
    console.log(`${this.name} is working`);
  }

  async eat(): Promise<void> {
    console.log(`${this.name} is eating`);
  }

  async sleep(): Promise<void> {
    console.log(`${this.name} is sleeping`);
  }

  getSalary(): number {
    return this.salary;
  }

  async takeVacation(days: number): Promise<void> {
    console.log(`${this.name} is taking ${days} days vacation`);
  }

  async submitReport(): Promise<void> {
    console.log(`${this.name} is submitting report`);
  }

  async attendMeeting(): Promise<void> {
    console.log(`${this.name} is attending meeting`);
  }
}

// Robot Worker เฉพาะที่จำเป็น
class RobotWorker implements Workable, Reportable, MeetingAttendable {
  constructor(
    private id: string,
    private powerLevel: number
  ) {}

  async work(): Promise<void> {
    console.log(`Robot ${this.id} is working`);
  }

  async submitReport(): Promise<void> {
    console.log(`Robot ${this.id} is submitting report`);
  }

  async attendMeeting(): Promise<void> {
    console.log(`Robot ${this.id} is attending meeting`);
  }
}

// Functions ที่ใช้ interface เฉพาะ
async function makeWork(worker: Workable): Promise<void> {
  await worker.work();
}

async function makeEat(worker: Eatable): Promise<void> {
  await worker.eat();
}

async function makeSleep(worker: Sleepable): Promise<void> {
  await worker.sleep();
}

function getWorkerSalary(worker: Payable): number {
  return worker.getSalary();
}

async function makeTakeVacation(worker: Vacationable, days: number): Promise<void> {
  await worker.takeVacation(days);
}

async function makeSubmitReport(worker: Reportable): Promise<void> {
  await worker.submitReport();
}

async function makeAttendMeeting(worker: MeetingAttendable): Promise<void> {
  await worker.attendMeeting();
}
```

### ข้อดีของโค้ดที่ดี:
1. **Client implement เฉพาะที่จำเป็น**: Robot ไม่ต้อง implement method ที่ไม่ได้ใช้
2. **แยกความรับผิดชอบ**: แต่ละ interface มีหน้าที่เดียว
3. **ขยายง่าย**: เพิ่ม interface ใหม่ได้โดยไม่กระทบส่วนอื่น
4. **ยืดหยุ่น**: สามารถผสมผสาน interface ได้ตามต้องการ

---

## ตัวอย่างการใช้งานจริง

```typescript
async function main(): Promise<void> {
  const human = new HumanWorker('John', 50000);
  const robot = new RobotWorker('R2D2', 100);

  // Human สามารถทำได้ทุกอย่าง
  await makeWork(human);
  await makeEat(human);
  await makeSleep(human);
  console.log(`Human salary: $${getWorkerSalary(human)}`);
  await makeTakeVacation(human, 5);
  await makeSubmitReport(human);
  await makeAttendMeeting(human);

  // Robot เฉพาะที่จำเป็น
  await makeWork(robot);
  await makeSubmitReport(robot);
  await makeAttendMeeting(robot);
  // await makeEat(robot); // จะ compile error เพราะ Robot ไม่ implement Eatable
}

main().catch(console.error);
```

---

## การทดสอบ (Testing)

```typescript
describe('Worker Tests', () => {
  test('Human worker has all capabilities', async () => {
    const human = new HumanWorker('John', 50000);

    await expect(makeWork(human)).resolves.not.toThrow();
    await expect(makeEat(human)).resolves.not.toThrow();
    await expect(makeSleep(human)).resolves.not.toThrow();
    expect(getWorkerSalary(human)).toBe(50000);
    await expect(makeTakeVacation(human, 5)).resolves.not.toThrow();
    await expect(makeSubmitReport(human)).resolves.not.toThrow();
    await expect(makeAttendMeeting(human)).resolves.not.toThrow();
  });

  test('Robot worker has limited capabilities', async () => {
    const robot = new RobotWorker('R2D2', 100);

    await expect(makeWork(robot)).resolves.not.toThrow();
    await expect(makeSubmitReport(robot)).resolves.not.toThrow();
    await expect(makeAttendMeeting(robot)).resolves.not.toThrow();
    // robot ไม่สามารถ eat, sleep, take_vacation ได้
  });

  test('Workable interface works with both workers', async () => {
    const human = new HumanWorker('John', 50000);
    const robot = new RobotWorker('R2D2', 100);

    await expect(makeWork(human)).resolves.not.toThrow();
    await expect(makeWork(robot)).resolves.not.toThrow();
  });
});
```

---

## สรุป

**ISP ใน TypeScript ช่วยให้:**
- Client implement เฉพาะ interface ที่จำเป็น
- แยกความรับผิดชอบได้ชัดเจน
- ง่ายต่อการขยายและดูแลรักษา
- ลดการพึ่งพากันระหว่างส่วนต่างๆ

**เมื่อไหร่ควรใช้:**
- เมื่อมี fat interface ที่มี method มากเกินไป
- เมื่อ client ถูกบังคับให้ implement method ที่ไม่ได้ใช้
- เมื่อต้องการความยืดหยุ่นในการผสมผสาน interface

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - การแยก interface เล็กๆ ช่วยให้ SRP ทำงานได้ดีขึ้น
- **[Liskov Substitution Principle](./03-liskov-substitution-principle.md)** - การแยก interface ช่วยให้ LSP ทำงานได้ดีขึ้น 