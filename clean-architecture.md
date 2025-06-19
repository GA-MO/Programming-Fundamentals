# Clean Architecture ใน Go

Clean Architecture เป็นแนวทางการออกแบบซอฟต์แวร์ที่แยกส่วนต่างๆ ออกจากกันอย่างชัดเจน ทำให้โค้ดมีความยืดหยุ่น ทดสอบง่าย และบำรุงรักษาได้ดี

## 📋 Table of Contents

- [โครงสร้างของ Clean Architecture](#โครงสร้างของ-clean-architecture)
- [Project Structure](#project-structure)
- [ตัวอย่างการใช้งานจริง: ระบบจัดการผู้ใช้](#ตัวอย่างการใช้งานจริง-ระบบจัดการผู้ใช้)
- [จุดเด่นของ Clean Architecture](#จุดเด่นของ-clean-architecture)
- [การเปรียบเทียบกับ SOLID Principles](#การเปรียบเทียบกับ-solid-principles)
- [ข้อดีและข้อเสีย](#ข้อดีและข้อเสีย)
- [เมื่อไหร่ควรใช้ Clean Architecture](#เมื่อไหร่ควรใช้-clean-architecture)
- [สรุป](#สรุป)

---

## โครงสร้างของ Clean Architecture

Clean Architecture แบ่งโค้ดออกเป็น **4 ชั้น (Layers)** ที่มีหน้าที่แตกต่างกัน โดยแต่ละชั้นจะพึ่งพาเฉพาะชั้นที่อยู่ด้านในเท่านั้น

### 🏗️ **4 ชั้นหลักของ Clean Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                    🚪 External Interfaces                   │
│  (Web API, CLI, Database, External Services)               │
├─────────────────────────────────────────────────────────────┤
│                    🔌 Interface Adapters                    │
│  (Controllers, Presenters, Gateways, Repositories)         │
├─────────────────────────────────────────────────────────────┤
│                    ⚙️ Application Business Rules            │
│  (Use Cases, Application Services, DTOs)                   │
├─────────────────────────────────────────────────────────────┤
│                    💼 Enterprise Business Rules             │
│  (Entities, Domain Services, Value Objects)                │
└─────────────────────────────────────────────────────────────┘
```

### 📋 **หน้าที่ของแต่ละชั้น**

#### 1. **Enterprise Business Rules (ชั้นในสุด)**
- **หน้าที่**: เก็บ business logic และ rules ที่สำคัญที่สุด
- **ตัวอย่าง**: User entity, business validation rules
- **ไม่ขึ้นต่อ**: ภายนอกใดๆ (database, web framework, etc.)

#### 2. **Application Business Rules**
- **หน้าที่**: จัดการ use cases และ workflow ของแอปพลิเคชัน
- **ตัวอย่าง**: การสร้าง user, การส่ง email
- **ขึ้นต่อ**: Enterprise Business Rules เท่านั้น

#### 3. **Interface Adapters**
- **หน้าที่**: แปลงข้อมูลระหว่าง external systems กับ application
- **ตัวอย่าง**: HTTP controllers, database repositories
- **ขึ้นต่อ**: Application Business Rules

#### 4. **External Interfaces (ชั้นนอกสุด)**
- **หน้าที่**: เชื่อมต่อกับ external systems
- **ตัวอย่าง**: Web server, database, email service
- **ขึ้นต่อ**: Interface Adapters

### 🔄 **Dependency Rule (กฎการพึ่งพา)**

**ลูกศรแสดงทิศทางการพึ่งพา:**
```
External Interfaces → Interface Adapters → Application → Enterprise
```

**กฎสำคัญ**: การพึ่งพาจะชี้เข้าด้านในเท่านั้น (Dependency Inversion)

### 💡 **ตัวอย่างการทำงาน**

สมมติว่าเราต้องการสร้างระบบจัดการผู้ใช้:

#### **ขั้นตอนที่ 1: รับข้อมูลจาก Web API**
```
Web Request → Controller → Use Case → Entity
```

#### **ขั้นตอนที่ 2: บันทึกข้อมูลลง Database**
```
Entity → Use Case → Repository → Database
```

#### **ขั้นตอนที่ 3: ส่ง Email**
```
Use Case → Email Service → SMTP Server
```

### 🎯 **ข้อดีของโครงสร้างนี้**

| ข้อดี | คำอธิบาย |
|-------|----------|
| **🔄 ยืดหยุ่น** | เปลี่ยน database, web framework ได้ง่าย |
| **🧪 ทดสอบง่าย** | ทดสอบแต่ละส่วนแยกกันได้ |
| **👥 ทีมทำงาน** | ทีมสามารถทำงานแยกกันได้ |
| **🔧 บำรุงรักษา** | แก้ไขโค้ดได้ง่าย ไม่กระทบส่วนอื่น |
| **📈 ขยายได้** | เพิ่ม features ใหม่ได้ง่าย |


### 🔧 **ตัวอย่างการใช้งานจริง**

#### **Domain Layer (ชั้นในสุด)**
```go
// internal/model/user.go
type User struct {
    ID        string
    Email     string
    Name      string
    Password  string
    CreatedAt time.Time
    UpdatedAt time.Time
}

// NewUser creates a new user with validation
func NewUser(email, name, password string) (*User, error) {
    if email == "" {
        return nil, errors.New("email cannot be empty")
    }
    // ... validation logic
}
```

#### **Application Layer**
```go
// internal/service/user_service.go
type UserService interface {
    CreateUser(email, name, password string) error
}

type UserService struct {
    userRepo     repository.UserRepository  // ขึ้นต่อ interface
    emailService service.EmailService    // ขึ้นต่อ interface
}

func (s *UserService) CreateUser(email, name, password string) error {
    // Business logic
    user, err := model.NewUser(email, name, password)
    if err != nil {
        return err
    }
    
    // Save to repository (ไม่รู้ว่าเป็น database อะไร)
    return s.userRepo.Save(user)
}
```

#### **Infrastructure Layer**
```go
// internal/repository/user_repository.go
type UserRepository interface {
    Save(user *model.User) error
}


// internal/repository/postgres_user_repository.go
type PostgresUserRepository struct {
    db *sql.DB
}

// Implement interface ที่ domain กำหนด
func (r *PostgresUserRepository) Save(user *model.User) error {
    // Database-specific code
    query := `INSERT INTO users (id, email, name) VALUES ($1, $2, $3)`
    _, err := r.db.Exec(query, user.ID, user.Email, user.Name)
    return err
}
```

#### **Interface Layer**
```go
// internal/handler/user_handler.go
type UserHandler struct {
    userService service.UserService  // ขึ้นต่อ user service interface
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // HTTP-specific code
    var req CreateUserRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // เรียกใช้ business logic
    err := h.userService.CreateUser(req.Email, req.Name, req.Password)
    // ... response handling
}
```

### 🎓 **สรุป**

Clean Architecture ช่วยให้เรา:

1. **แยกส่วนชัดเจน** - แต่ละส่วนมีหน้าที่เฉพาะ
2. **พึ่งพาถูกทิศทาง** - จากนอกเข้าข้างใน
3. **ทดสอบง่าย** - ทดสอบแต่ละส่วนแยกกัน
4. **เปลี่ยนได้ง่าย** - เปลี่ยน database, framework ได้
5. **ขยายได้ง่าย** - เพิ่ม features ใหม่ได้

**หลักการสำคัญ**: **"Dependency Inversion"** - ขึ้นต่อ abstractions ไม่ใช่ concretions

## Project Structure

### โครงสร้างแบบง่าย (Simple Clean Architecture)

```
user-service/
├── cmd/
│   └── main.go                     # Entry point
├── internal/
│   ├── model/                      # Business entities (Domain Layer)
│   │   └── user.go                 # User model และ domain interfaces
│   ├── service/                    # Business logic (Application Layer)
│   │   ├── user_service.go         # User business operations
│   │   ├── email_service.go        # Email business operations
│   │   └── interfaces.go           # Application service interfaces
│   ├── repository/                 # Data access (Infrastructure Layer)
│   │   ├── user_repository.go      # In-memory implementation
│   │   └── interfaces.go           # Repository interfaces (domain contracts)
│   ├── handler/                    # HTTP handlers (Interface Layer)
│   │   ├── user_handler.go         # User HTTP handlers
│   │   ├── router.go               # Route definitions
│   │   └── interfaces.go           # Handler interfaces
│   └── middleware/                 # HTTP middleware
│       ├── auth.go                 # Authentication middleware
│       ├── logging.go              # Logging middleware
├── pkg/                            # Shared utilities
│   ├── logger.go
│   ├── validator.go
│   └── utils.go
├── configs/
│   ├── config.yaml
├── go.mod
├── go.sum
├── Makefile                        # Build and test commands
├── README.md
└── .gitignore
```

### Testing Strategy

โครงสร้างการทดสอบในโปรเจกต์นี้ปฏิบัติตาม Clean Architecture principles โดยแยก testing ออกเป็นหลายระดับ:

- **Unit Tests** - ทดสอบแต่ละ component แยกกัน
- **Integration Tests** - ทดสอบการทำงานร่วมกัน
- **Mock Objects** - สำหรับทดสอบ dependencies
- **Test Fixtures** - Test data และ utilities

**สำหรับรายละเอียดเพิ่มเติมเกี่ยวกับการทดสอบ โปรดดู:**
📋 **[Clean Architecture Project Structure for Testing](./clean-architecture-project-structure-for-testing.md)** - โครงสร้างโปรเจกต์และการทดสอบแบบครบถ้วน

## จุดเด่นของ Clean Architecture

### 1. **Testability (ทดสอบง่าย)**
```go
// service/user_service_test.go
package service

import (
    "testing"
    "your-project/internal/repository"
    "your-project/internal/service"
)

func TestUserService_CreateUser(t *testing.T) {
    // Arrange
    userRepo := repository.NewMemoryUserRepository()
    emailService := email.NewEmailService("localhost", 587)
    userService := NewUserService(userRepo, emailService)

    // Act
    user, err := userService.CreateUser("test@example.com", "Test User", "password123")

    // Assert
    if err != nil {
        t.Errorf("Expected no error, got %v", err)
    }
    if user == nil {
        t.Error("Expected user to be created")
    }
    if user.Email != "test@example.com" {
        t.Errorf("Expected email 'test@example.com', got %s", user.Email)
    }
}
```

### 2. **Framework Independence**
- สามารถเปลี่ยนจาก `net/http` เป็น `gin`, `echo`, หรือ `fiber` ได้ง่าย
- Business logic ไม่ขึ้นต่อ framework

### 3. **Database Independence**
```go
// repository/postgres_user_repository.go
package repository

import (
    "database/sql"
    "your-project/internal/model"
    "your-project/internal/repository"
)

type PostgresUserRepository struct {
    db *sql.DB
}

func NewPostgresUserRepository(db *sql.DB) repository.UserRepository {
    return &PostgresUserRepository{db: db}
}

func (r *PostgresUserRepository) Save(user *model.User) error {
    query := `INSERT INTO users (id, email, name, password, created_at, updated_at) 
              VALUES ($1, $2, $3, $4, $5, $6) 
              ON CONFLICT (id) DO UPDATE SET 
              email = $2, name = $3, password = $4, updated_at = $6`
    
    _, err := r.db.Exec(query, user.ID, user.Email, user.Name, 
                       user.Password, user.CreatedAt, user.UpdatedAt)
    return err
}

// ... other methods
```

### 4. **Separation of Concerns**
- **Domain**: Business rules และ entities
- **Application**: Use cases และ application services
- **Infrastructure**: External concerns (database, HTTP, etc.)
- **Interfaces**: Adapters สำหรับ external systems

## การเปรียบเทียบกับ SOLID Principles

Clean Architecture สอดคล้องกับ SOLID Principles:

1. **SRP**: แต่ละ layer มีหน้าที่เดียว
2. **OCP**: สามารถขยายได้โดยไม่แก้ไข code เดิม
3. **LSP**: Repository implementations สามารถแทนที่กันได้
4. **ISP**: Interfaces เล็กและเฉพาะเจาะจง
5. **DIP**: ขึ้นต่อ abstractions ไม่ใช่ concretions

## ข้อดีและข้อเสีย

### ข้อดี
- ✅ ทดสอบง่าย
- ✅ บำรุงรักษาได้ดี
- ✅ ยืดหยุ่นสูง
- ✅ Framework independent
- ✅ Database independent
- ✅ Team collaboration ดี

### ข้อเสีย
- ❌ Complexity สูง
- ❌ Learning curve สูง
- ❌ Over-engineering สำหรับโปรเจกต์เล็ก
- ❌ Performance overhead (minimal)

## เมื่อไหร่ควรใช้ Clean Architecture

### ควรใช้เมื่อ:
- ระบบขนาดใหญ่
- มี business logic ซับซ้อน
- ต้องการ maintainability สูง
- ทีมขนาดใหญ่
- ต้องการ testability สูง

### ไม่ควรใช้เมื่อ:
- ระบบขนาดเล็ก
- Prototype หรือ MVP
- ไม่มี business logic ซับซ้อน
- ต้องการความเร็วในการพัฒนา

## สรุป

Clean Architecture ใน Go ช่วยให้เราเขียนโค้ดที่มีโครงสร้างชัดเจน ทดสอบง่าย และบำรุงรักษาได้ดี โดยแยกส่วนต่างๆ ตามหน้าที่และใช้ dependency injection เพื่อลดการพึ่งพา

---

**References:**
- [Clean Architecture Project Structure for Testing](./clean-architecture-project-structure-for-testing.md) - โครงสร้างโปรเจกต์และการทดสอบแบบครบถ้วน
- [SOLID Design Principles](./solid/solid-design-principles.md)
- [Clean Code](./clean-code.md)
- [Code Smells](./code-smells.md)
