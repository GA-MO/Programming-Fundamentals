# Interface Segregation Principle (ISP)

## ความหมาย
**"Client ไม่ควรถูกบังคับให้ขึ้นต่อ interface ที่ไม่ได้ใช้"** - หมายความว่า interface ควรเล็กลงและเฉพาะเจาะจง ไม่ใช่ interface ใหญ่ที่บังคับให้ client ต้อง implement method ที่ไม่ได้ใช้

## หลักการ
- **Small Interfaces**: Interface ควรมี method น้อยๆ และเฉพาะเจาะจง
- **Client-Specific**: แต่ละ client ควรขึ้นต่อ interface ที่ต้องการเท่านั้น
- **No Forced Dependencies**: ไม่บังคับให้ implement method ที่ไม่ได้ใช้

---

## ❌ ตัวอย่างที่ไม่ดี (Bad Example)

### Interface ใหญ่เกินไป (Fat Interface)

```go
// Worker interface ที่ใหญ่เกินไป
type Worker interface {
    Work() string
    Eat() string
    Sleep() string
    GetSalary() float64
    TakeVacation() string
    GetPerformance() string
    SubmitReport() string
    AttendMeeting() string
    Code() string
    Design() string
    Test() string
    Deploy() string
}

// HumanWorker - คนทำงานทั่วไป
type HumanWorker struct {
    name string
}

func (h *HumanWorker) Work() string {
    return fmt.Sprintf("%s is working", h.name)
}

func (h *HumanWorker) Eat() string {
    return fmt.Sprintf("%s is eating", h.name)
}

func (h *HumanWorker) Sleep() string {
    return fmt.Sprintf("%s is sleeping", h.name)
}

func (h *HumanWorker) GetSalary() float64 {
    return 50000.0
}

func (h *HumanWorker) TakeVacation() string {
    return fmt.Sprintf("%s is on vacation", h.name)
}

func (h *HumanWorker) GetPerformance() string {
    return "Good performance"
}

func (h *HumanWorker) SubmitReport() string {
    return fmt.Sprintf("%s submitted report", h.name)
}

func (h *HumanWorker) AttendMeeting() string {
    return fmt.Sprintf("%s attended meeting", h.name)
}

func (h *HumanWorker) Code() string {
    return fmt.Sprintf("%s is coding", h.name)
}

func (h *HumanWorker) Design() string {
    return fmt.Sprintf("%s is designing", h.name)
}

func (h *HumanWorker) Test() string {
    return fmt.Sprintf("%s is testing", h.name)
}

func (h *HumanWorker) Deploy() string {
    return fmt.Sprintf("%s is deploying", h.name)
}

// RobotWorker - หุ่นยนต์ที่ทำได้แค่บางอย่าง
type RobotWorker struct {
    id string
}

func (r *RobotWorker) Work() string {
    return fmt.Sprintf("Robot %s is working", r.id)
}

func (r *RobotWorker) Eat() string {
    return fmt.Sprintf("Robot %s doesn't eat", r.id) // ไม่เหมาะสม!
}

func (r *RobotWorker) Sleep() string {
    return fmt.Sprintf("Robot %s doesn't sleep", r.id) // ไม่เหมาะสม!
}

func (r *RobotWorker) GetSalary() float64 {
    return 0.0 // หุ่นยนต์ไม่ได้รับเงินเดือน!
}

func (r *RobotWorker) TakeVacation() string {
    return fmt.Sprintf("Robot %s doesn't take vacation", r.id) // ไม่เหมาะสม!
}

func (r *RobotWorker) GetPerformance() string {
    return "Excellent performance"
}

func (r *RobotWorker) SubmitReport() string {
    return fmt.Sprintf("Robot %s submitted report", r.id)
}

func (r *RobotWorker) AttendMeeting() string {
    return fmt.Sprintf("Robot %s attended meeting", r.id)
}

func (r *RobotWorker) Code() string {
    return fmt.Sprintf("Robot %s is coding", r.id)
}

func (r *RobotWorker) Design() string {
    return fmt.Sprintf("Robot %s is designing", r.id)
}

func (r *RobotWorker) Test() string {
    return fmt.Sprintf("Robot %s is testing", r.id)
}

func (r *RobotWorker) Deploy() string {
    return fmt.Sprintf("Robot %s is deploying", r.id)
}

// WorkManager ที่ใช้งาน
type WorkManager struct{}

func (m *WorkManager) AssignWork(worker Worker) {
    fmt.Println(worker.Work())
    fmt.Println(worker.GetPerformance())
}

func (m *WorkManager) PaySalary(worker Worker) {
    salary := worker.GetSalary()
    fmt.Printf("Paid $%.2f to worker\n", salary)
    // ปัญหา: หุ่นยนต์จะได้รับเงินเดือน $0.00 ซึ่งไม่เหมาะสม
}
```

---

## ✅ ตัวอย่างที่ดี (Good Example)

### แยก Interface ตามความสามารถ

```go
// แยก interface ตามความสามารถ

// Basic Worker
type Worker interface {
    Work() string
    GetPerformance() string
}

// Living Being
type LivingBeing interface {
    Eat() string
    Sleep() string
}

// Paid Worker
type PaidWorker interface {
    Worker
    GetSalary() float64
    TakeVacation() string
}

// Reportable Worker
type ReportableWorker interface {
    Worker
    SubmitReport() string
    AttendMeeting() string
}

// Software Developer
type SoftwareDeveloper interface {
    Worker
    Code() string
    Test() string
    Deploy() string
}

// HumanWorker - คนทำงานทั่วไป
type HumanWorker struct {
    name string
}

func (h *HumanWorker) Work() string {
    return fmt.Sprintf("%s is working", h.name)
}

func (h *HumanWorker) GetPerformance() string {
    return "Good performance"
}

func (h *HumanWorker) Eat() string {
    return fmt.Sprintf("%s is eating", h.name)
}

func (h *HumanWorker) Sleep() string {
    return fmt.Sprintf("%s is sleeping", h.name)
}

func (h *HumanWorker) GetSalary() float64 {
    return 50000.0
}

func (h *HumanWorker) TakeVacation() string {
    return fmt.Sprintf("%s is on vacation", h.name)
}

func (h *HumanWorker) SubmitReport() string {
    return fmt.Sprintf("%s submitted report", h.name)
}

func (h *HumanWorker) AttendMeeting() string {
    return fmt.Sprintf("%s attended meeting", h.name)
}

func (h *HumanWorker) Code() string {
    return fmt.Sprintf("%s is coding", h.name)
}

func (h *HumanWorker) Test() string {
    return fmt.Sprintf("%s is testing", h.name)
}

func (h *HumanWorker) Deploy() string {
    return fmt.Sprintf("%s is deploying", h.name)
}

// RobotWorker - หุ่นยนต์ที่ทำได้แค่บางอย่าง
type RobotWorker struct {
    id string
}

func (r *RobotWorker) Work() string {
    return fmt.Sprintf("Robot %s is working", r.id)
}

func (r *RobotWorker) GetPerformance() string {
    return "Excellent performance"
}

func (r *RobotWorker) SubmitReport() string {
    return fmt.Sprintf("Robot %s submitted report", r.id)
}

func (r *RobotWorker) AttendMeeting() string {
    return fmt.Sprintf("Robot %s attended meeting", r.id)
}

func (r *RobotWorker) Code() string {
    return fmt.Sprintf("Robot %s is coding", r.id)
}

func (r *RobotWorker) Test() string {
    return fmt.Sprintf("Robot %s is testing", r.id)
}

func (r *RobotWorker) Deploy() string {
    return fmt.Sprintf("Robot %s is deploying", r.id)
}

// WorkManager ที่ใช้งานเฉพาะ interface ที่ต้องการ
type WorkManager struct{}

func (m *WorkManager) AssignWork(worker Worker) {
    fmt.Println(worker.Work())
    fmt.Println(worker.GetPerformance())
}

func (m *WorkManager) PaySalary(worker PaidWorker) {
    salary := worker.GetSalary()
    fmt.Printf("Paid $%.2f to worker\n", salary)
}

func (m *WorkManager) ManageReports(worker ReportableWorker) {
    fmt.Println(worker.SubmitReport())
    fmt.Println(worker.AttendMeeting())
}

func (m *WorkManager) ManageDevelopment(developer SoftwareDeveloper) {
    fmt.Println(developer.Code())
    fmt.Println(developer.Test())
    fmt.Println(developer.Deploy())
}
```

---

## ตัวอย่างการใช้งานจริง

```go
func main() {
    workManager := &WorkManager{}
    
    // ใช้งานกับ Human Worker
    human := &HumanWorker{name: "John"}
    workManager.AssignWork(human)           // Worker interface
    workManager.PaySalary(human)            // PaidWorker interface
    workManager.ManageReports(human)        // ReportableWorker interface
    workManager.ManageDevelopment(human)    // SoftwareDeveloper interface
    
    // ใช้งานกับ Robot Worker
    robot := &RobotWorker{id: "R001"}
    workManager.AssignWork(robot)           // Worker interface
    workManager.ManageReports(robot)        // ReportableWorker interface
    workManager.ManageDevelopment(robot)    // SoftwareDeveloper interface
    // workManager.PaySalary(robot)         // Compile error! ดีแล้ว
}
```

---

## การใช้ Interface Composition

```go
// ใช้ interface composition เพื่อสร้าง interface ที่ซับซ้อนขึ้น

// Basic Worker
type Worker interface {
    Work() string
    GetPerformance() string
}

// Living Being
type LivingBeing interface {
    Eat() string
    Sleep() string
}

// Paid Worker = Worker + Living Being + Salary
type PaidWorker interface {
    Worker
    LivingBeing
    GetSalary() float64
    TakeVacation() string
}

// Software Developer = Worker + Development Skills
type SoftwareDeveloper interface {
    Worker
    Code() string
    Test() string
    Deploy() string
}

// Full Stack Developer = Software Developer + Design Skills
type FullStackDeveloper interface {
    SoftwareDeveloper
    Design() string
}

// Senior Developer = Full Stack Developer + Leadership
type SeniorDeveloper interface {
    FullStackDeveloper
    Mentor() string
    ReviewCode() string
}

// Human Developer - implement SeniorDeveloper
type HumanDeveloper struct {
    name string
}

func (h *HumanDeveloper) Work() string {
    return fmt.Sprintf("%s is working", h.name)
}

func (h *HumanDeveloper) GetPerformance() string {
    return "Excellent performance"
}

func (h *HumanDeveloper) Eat() string {
    return fmt.Sprintf("%s is eating", h.name)
}

func (h *HumanDeveloper) Sleep() string {
    return fmt.Sprintf("%s is sleeping", h.name)
}

func (h *HumanDeveloper) GetSalary() float64 {
    return 80000.0
}

func (h *HumanDeveloper) TakeVacation() string {
    return fmt.Sprintf("%s is on vacation", h.name)
}

func (h *HumanDeveloper) Code() string {
    return fmt.Sprintf("%s is coding", h.name)
}

func (h *HumanDeveloper) Test() string {
    return fmt.Sprintf("%s is testing", h.name)
}

func (h *HumanDeveloper) Deploy() string {
    return fmt.Sprintf("%s is deploying", h.name)
}

func (h *HumanDeveloper) Design() string {
    return fmt.Sprintf("%s is designing", h.name)
}

func (h *HumanDeveloper) Mentor() string {
    return fmt.Sprintf("%s is mentoring", h.name)
}

func (h *HumanDeveloper) ReviewCode() string {
    return fmt.Sprintf("%s is reviewing code", h.name)
}
```

---

## การทดสอบ (Testing)

```go
func TestWorkManager_AssignWork(t *testing.T) {
    manager := &WorkManager{}
    
    // Test with Human Worker
    human := &HumanWorker{name: "John"}
    manager.AssignWork(human) // ควรทำงานได้
    
    // Test with Robot Worker
    robot := &RobotWorker{id: "R001"}
    manager.AssignWork(robot) // ควรทำงานได้
}

func TestWorkManager_PaySalary(t *testing.T) {
    manager := &WorkManager{}
    
    // Test with Human Worker (PaidWorker)
    human := &HumanWorker{name: "John"}
    manager.PaySalary(human) // ควรทำงานได้
    
    // Test with Robot Worker (ไม่ใช่ PaidWorker)
    robot := &RobotWorker{id: "R001"}
    // manager.PaySalary(robot) // Compile error - ดีแล้ว!
}
```

---

## สรุป

**ISP ช่วยให้:**
- Interface เล็กลงและเฉพาะเจาะจง
- Client ไม่ต้อง implement method ที่ไม่ได้ใช้
- ลดการพึ่งพากันที่ไม่จำเป็น
- ง่ายต่อการทดสอบและดูแลรักษา
- เพิ่มความยืดหยุ่นในการใช้งาน

**เมื่อไหร่ควรใช้:**
- เมื่อ interface มี method มากเกินไป
- เมื่อ client ต้อง implement method ที่ไม่ได้ใช้
- เมื่อต้องการลดการพึ่งพากันระหว่างส่วนต่างๆ

**หลักการสำคัญ:**
- **YAGNI** (You Aren't Gonna Need It) - ไม่เพิ่ม method ที่ยังไม่ต้องการ
- **Interface Composition** - รวม interface เล็กๆ เป็น interface ใหญ่
- **Client-Specific Interfaces** - แต่ละ client ขึ้นต่อ interface ที่ต้องการเท่านั้น

---

## อ้างอิงไปยังหลักการอื่นๆ

- **[Single Responsibility Principle](./01-single-responsibility-principle.md)** - Interface เล็กๆ มีหน้าที่เดียว
- **[Liskov Substitution Principle](./03-liskov-substitution-principle.md)** - Interface เล็กๆ ช่วยให้ LSP ทำงานได้ดีขึ้น
- **[Dependency Inversion Principle](./05-dependency-inversion-principle.md)** - การใช้ interface เล็กๆ ช่วยให้ DIP ทำงานได้ดีขึ้น
```