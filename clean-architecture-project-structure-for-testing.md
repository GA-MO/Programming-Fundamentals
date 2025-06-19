# Clean Architecture Project Structure for Testing

## Testing Strategy ด้วย Testify, SQLMock และ Go-Fiber

โครงสร้างการทดสอบที่ใช้ modern Go testing tools:

- **Testify** - Testing framework ที่สะดวกและอ่านง่าย
- **SQLMock** - Mock SQL database สำหรับทดสอบ
- **Go-Fiber** - Fast HTTP framework
- **Clean Architecture** - แยก layers อย่างชัดเจน

## โครงสร้างโปรเจกต์

```
user-service/
├── cmd/
│   └── main.go                     # Entry point
├── internal/
│   ├── domain/                     # Business entities
│   │   ├── user.go                 
│   │   ├── user_test.go            
│   │   └── interfaces.go           # Repository interfaces
│   ├── usecase/                    # Business logic
│   │   ├── user_usecase.go         
│   │   └── user_usecase_test.go    
│   ├── repository/                 # Data access (PostgreSQL only)
│   │   ├── user_repository.go      
│   │   └── user_repository_test.go # Uses SQLMock
│   └── handler/                    # HTTP handlers (Fiber)
│       ├── user_handler.go         
│       └── user_handler_test.go    
├── pkg/                            # Shared utilities
│   └── database/
│       └── postgres.go
├── test/                           # Test utilities
│   └── mocks/                      # Generated mocks
├── go.mod
└── go.sum
```

## Dependencies

```go
// go.mod
module user-service

go 1.21

require (
    github.com/gofiber/fiber/v2 v2.50.0
    github.com/stretchr/testify v1.8.4
    github.com/DATA-DOG/go-sqlmock v1.5.0
    github.com/lib/pq v1.10.9
    github.com/google/uuid v1.3.1
)
```

## Domain Layer

### User Entity
```go
// internal/domain/user.go
package domain

import (
    "errors"
    "time"
    "github.com/google/uuid"
)

type User struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    CreatedAt time.Time `json:"createdAt"`
}

func NewUser(email, name string) (*User, error) {
    if email == "" {
        return nil, errors.New("email required")
    }
    if name == "" {
        return nil, errors.New("name required")
    }
    
    return &User{
        ID:        uuid.New().String(),
        Email:     email,
        Name:      name,
        CreatedAt: time.Now(),
    }, nil
}
```

### Repository Interface
```go
// internal/domain/interfaces.go
package domain

type UserRepository interface {
    Create(user *User) error
    GetByID(id string) (*User, error)
    GetByEmail(email string) (*User, error)
}
```

### Unit Tests with Testify
```go
// internal/domain/user_test.go
package domain

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestNewUser(t *testing.T) {
    t.Run("valid user", func(t *testing.T) {
        user, err := NewUser("test@example.com", "Test User")
        
        require.NoError(t, err)
        assert.NotNil(t, user)
        assert.Equal(t, "test@example.com", user.Email)
        assert.Equal(t, "Test User", user.Name)
        assert.NotEmpty(t, user.ID)
        assert.False(t, user.CreatedAt.IsZero())
    })

    t.Run("empty email", func(t *testing.T) {
        user, err := NewUser("", "Test User")
        
        assert.Error(t, err)
        assert.Nil(t, user)
        assert.Equal(t, "email required", err.Error())
    })

    t.Run("empty name", func(t *testing.T) {
        user, err := NewUser("test@example.com", "")
        
        assert.Error(t, err)
        assert.Nil(t, user)
        assert.Equal(t, "name required", err.Error())
    })
}
```

## Usecase Layer

### User Usecase
```go
// internal/usecase/user_usecase.go
package usecase

import (
    "user-service/internal/domain"
)

type UserUsecase struct {
    userRepo domain.UserRepository
}

func NewUserUsecase(userRepo domain.UserRepository) *UserUsecase {
    return &UserUsecase{
        userRepo: userRepo,
    }
}

func (u *UserUsecase) CreateUser(email, name string) (*domain.User, error) {
    // Check if user exists
    existingUser, _ := u.userRepo.GetByEmail(email)
    if existingUser != nil {
        return nil, errors.New("email already exists")
    }

    // Create new user
    user, err := domain.NewUser(email, name)
    if err != nil {
        return nil, err
    }

    // Save to repository
    if err := u.userRepo.Create(user); err != nil {
        return nil, err
    }

    return user, nil
}

func (u *UserUsecase) GetUser(id string) (*domain.User, error) {
    return u.userRepo.GetByID(id)
}
```

### Usecase Tests with Mocks
```go
// internal/usecase/user_usecase_test.go
package usecase

import (
    "errors"
    "testing"
    "user-service/internal/domain"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"
)

// MockUserRepository implements domain.UserRepository
type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Create(user *domain.User) error {
    args := m.Called(user)
    return args.Error(0)
}

func (m *MockUserRepository) GetByID(id string) (*domain.User, error) {
    args := m.Called(id)
    return args.Get(0).(*domain.User), args.Error(1)
}

func (m *MockUserRepository) GetByEmail(email string) (*domain.User, error) {
    args := m.Called(email)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.User), args.Error(1)
}

func TestUserUsecase_CreateUser(t *testing.T) {
    t.Run("success", func(t *testing.T) {
        mockRepo := new(MockUserRepository)
        usecase := NewUserUsecase(mockRepo)

        // Mock: email doesn't exist
        mockRepo.On("GetByEmail", "test@example.com").Return(nil, nil)
        mockRepo.On("Create", mock.AnythingOfType("*domain.User")).Return(nil)

        user, err := usecase.CreateUser("test@example.com", "Test User")

        require.NoError(t, err)
        assert.Equal(t, "test@example.com", user.Email)
        assert.Equal(t, "Test User", user.Name)
        mockRepo.AssertExpectations(t)
    })

    t.Run("duplicate email", func(t *testing.T) {
        mockRepo := new(MockUserRepository)
        usecase := NewUserUsecase(mockRepo)

        existingUser := &domain.User{Email: "test@example.com"}
        mockRepo.On("GetByEmail", "test@example.com").Return(existingUser, nil)

        user, err := usecase.CreateUser("test@example.com", "Test User")

        assert.Error(t, err)
        assert.Nil(t, user)
        assert.Equal(t, "email already exists", err.Error())
        mockRepo.AssertExpectations(t)
    })
}
```

## Repository Layer with SQLMock

### PostgreSQL Repository
```go
// internal/repository/user_repository.go
package repository

import (
    "database/sql"
    "user-service/internal/domain"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(user *domain.User) error {
    query := `INSERT INTO users (id, email, name, created_at) VALUES ($1, $2, $3, $4)`
    _, err := r.db.Exec(query, user.ID, user.Email, user.Name, user.CreatedAt)
    return err
}

func (r *UserRepository) GetByID(id string) (*domain.User, error) {
    var user domain.User
    query := `SELECT id, email, name, created_at FROM users WHERE id = $1`
    
    err := r.db.QueryRow(query, id).Scan(&user.ID, &user.Email, &user.Name, &user.CreatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, nil
        }
        return nil, err
    }
    return &user, nil
}

func (r *UserRepository) GetByEmail(email string) (*domain.User, error) {
    var user domain.User
    query := `SELECT id, email, name, created_at FROM users WHERE email = $1`
    
    err := r.db.QueryRow(query, email).Scan(&user.ID, &user.Email, &user.Name, &user.CreatedAt)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, nil
        }
        return nil, err
    }
    return &user, nil
}
```

### Repository Tests with SQLMock
```go
// internal/repository/user_repository_test.go
package repository

import (
    "testing"
    "time"
    "user-service/internal/domain"
    
    "github.com/DATA-DOG/go-sqlmock"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUserRepository_Create(t *testing.T) {
    db, mock, err := sqlmock.New()
    require.NoError(t, err)
    defer db.Close()

    repo := NewUserRepository(db)
    user := &domain.User{
        ID:        "123",
        Email:     "test@example.com",
        Name:      "Test User",
        CreatedAt: time.Now(),
    }

    mock.ExpectExec(`INSERT INTO users`).
        WithArgs(user.ID, user.Email, user.Name, user.CreatedAt).
        WillReturnResult(sqlmock.NewResult(1, 1))

    err = repo.Create(user)

    assert.NoError(t, err)
    assert.NoError(t, mock.ExpectationsWereMet())
}

func TestUserRepository_GetByID(t *testing.T) {
    db, mock, err := sqlmock.New()
    require.NoError(t, err)
    defer db.Close()

    repo := NewUserRepository(db)
    expectedUser := &domain.User{
        ID:        "123",
        Email:     "test@example.com",
        Name:      "Test User",
        CreatedAt: time.Now(),
    }

    rows := sqlmock.NewRows([]string{"id", "email", "name", "created_at"}).
        AddRow(expectedUser.ID, expectedUser.Email, expectedUser.Name, expectedUser.CreatedAt)

    mock.ExpectQuery(`SELECT (.+) FROM users WHERE id`).
        WithArgs("123").
        WillReturnRows(rows)

    user, err := repo.GetByID("123")

    require.NoError(t, err)
    assert.Equal(t, expectedUser.ID, user.ID)
    assert.Equal(t, expectedUser.Email, user.Email)
    assert.Equal(t, expectedUser.Name, user.Name)
    assert.NoError(t, mock.ExpectationsWereMet())
}

func TestUserRepository_GetByID_NotFound(t *testing.T) {
    db, mock, err := sqlmock.New()
    require.NoError(t, err)
    defer db.Close()

    repo := NewUserRepository(db)

    mock.ExpectQuery(`SELECT (.+) FROM users WHERE id`).
        WithArgs("999").
        WillReturnRows(sqlmock.NewRows([]string{"id", "email", "name", "created_at"}))

    user, err := repo.GetByID("999")

    assert.NoError(t, err)
    assert.Nil(t, user)
    assert.NoError(t, mock.ExpectationsWereMet())
}
```

## Handler Layer with Go-Fiber

### Fiber Handler
```go
// internal/handler/user_handler.go
package handler

import (
    "user-service/internal/usecase"
    "github.com/gofiber/fiber/v2"
)

type UserHandler struct {
    userUsecase *usecase.UserUsecase
}

type CreateUserRequest struct {
    Email string `json:"email" validate:"required,email"`
    Name  string `json:"name" validate:"required"`
}

func NewUserHandler(userUsecase *usecase.UserUsecase) *UserHandler {
    return &UserHandler{
        userUsecase: userUsecase,
    }
}

func (h *UserHandler) CreateUser(c *fiber.Ctx) error {
    var req CreateUserRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
    }

    user, err := h.userUsecase.CreateUser(req.Email, req.Name)
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": err.Error()})
    }

    return c.Status(201).JSON(user)
}

func (h *UserHandler) GetUser(c *fiber.Ctx) error {
    id := c.Params("id")
    if id == "" {
        return c.Status(400).JSON(fiber.Map{"error": "ID required"})
    }

    user, err := h.userUsecase.GetUser(id)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": err.Error()})
    }
    if user == nil {
        return c.Status(404).JSON(fiber.Map{"error": "User not found"})
    }

    return c.JSON(user)
}

func (h *UserHandler) SetupRoutes(app *fiber.App) {
    api := app.Group("/api/v1")
    api.Post("/users", h.CreateUser)
    api.Get("/users/:id", h.GetUser)
}
```

### Handler Tests with Fiber
```go
// internal/handler/user_handler_test.go
package handler

import (
    "bytes"
    "encoding/json"
    "net/http/httptest"
    "testing"
    "user-service/internal/domain"
    
    "github.com/gofiber/fiber/v2"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"
)

type MockUserUsecase struct {
    mock.Mock
}

func (m *MockUserUsecase) CreateUser(email, name string) (*domain.User, error) {
    args := m.Called(email, name)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.User), args.Error(1)
}

func (m *MockUserUsecase) GetUser(id string) (*domain.User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.User), args.Error(1)
}

func TestUserHandler_CreateUser(t *testing.T) {
    t.Run("success", func(t *testing.T) {
        mockUsecase := new(MockUserUsecase)
        handler := NewUserHandler(mockUsecase)
        
        app := fiber.New()
        handler.SetupRoutes(app)

        expectedUser := &domain.User{
            ID:    "123",
            Email: "test@example.com",
            Name:  "Test User",
        }

        mockUsecase.On("CreateUser", "test@example.com", "Test User").
            Return(expectedUser, nil)

        reqBody := CreateUserRequest{
            Email: "test@example.com",
            Name:  "Test User",
        }
        jsonBody, _ := json.Marshal(reqBody)

        req := httptest.NewRequest("POST", "/api/v1/users", bytes.NewBuffer(jsonBody))
        req.Header.Set("Content-Type", "application/json")
        resp, err := app.Test(req)

        require.NoError(t, err)
        assert.Equal(t, 201, resp.StatusCode)
        mockUsecase.AssertExpectations(t)
    })

    t.Run("invalid request", func(t *testing.T) {
        mockUsecase := new(MockUserUsecase)
        handler := NewUserHandler(mockUsecase)
        
        app := fiber.New()
        handler.SetupRoutes(app)

        req := httptest.NewRequest("POST", "/api/v1/users", bytes.NewBuffer([]byte("invalid json")))
        req.Header.Set("Content-Type", "application/json")
        resp, err := app.Test(req)

        require.NoError(t, err)
        assert.Equal(t, 400, resp.StatusCode)
    })
}

func TestUserHandler_GetUser(t *testing.T) {
    t.Run("success", func(t *testing.T) {
        mockUsecase := new(MockUserUsecase)
        handler := NewUserHandler(mockUsecase)
        
        app := fiber.New()
        handler.SetupRoutes(app)

        expectedUser := &domain.User{
            ID:    "123",
            Email: "test@example.com",
            Name:  "Test User",
        }

        mockUsecase.On("GetUser", "123").Return(expectedUser, nil)

        req := httptest.NewRequest("GET", "/api/v1/users/123", nil)
        resp, err := app.Test(req)

        require.NoError(t, err)
        assert.Equal(t, 200, resp.StatusCode)
        mockUsecase.AssertExpectations(t)
    })

    t.Run("user not found", func(t *testing.T) {
        mockUsecase := new(MockUserUsecase)
        handler := NewUserHandler(mockUsecase)
        
        app := fiber.New()
        handler.SetupRoutes(app)

        mockUsecase.On("GetUser", "999").Return(nil, nil)

        req := httptest.NewRequest("GET", "/api/v1/users/999", nil)
        resp, err := app.Test(req)

        require.NoError(t, err)
        assert.Equal(t, 404, resp.StatusCode)
        mockUsecase.AssertExpectations(t)
    })
}
```

## Integration Test

```go
// test/integration_test.go
package test

import (
    "bytes"
    "encoding/json"
    "net/http/httptest"
    "testing"
    
    "user-service/internal/domain"
    "user-service/internal/handler"
    "user-service/internal/usecase"
    
    "github.com/gofiber/fiber/v2"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"
)

// MockUserRepository for integration tests
type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) Create(user *domain.User) error {
    args := m.Called(user)
    return args.Error(0)
}

func (m *MockUserRepository) GetByID(id string) (*domain.User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.User), args.Error(1)
}

func (m *MockUserRepository) GetByEmail(email string) (*domain.User, error) {
    args := m.Called(email)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.User), args.Error(1)
}

func TestUserAPI_Integration(t *testing.T) {
    // Setup with mocked repository
    mockRepo := new(MockUserRepository)
    userUsecase := usecase.NewUserUsecase(mockRepo)
    userHandler := handler.NewUserHandler(userUsecase)
    
    app := fiber.New()
    userHandler.SetupRoutes(app)

    // Mock repository behavior
    expectedUser := &domain.User{
        ID:    "integration-123",
        Email: "integration@test.com", 
        Name:  "Integration Test",
    }
    
    mockRepo.On("GetByEmail", "integration@test.com").Return(nil, nil)
    mockRepo.On("Create", mock.AnythingOfType("*domain.User")).Return(nil)
    mockRepo.On("GetByID", mock.AnythingOfType("string")).Return(expectedUser, nil)

    // Create user
    createReq := handler.CreateUserRequest{
        Email: "integration@test.com",
        Name:  "Integration Test",
    }
    jsonBody, _ := json.Marshal(createReq)

    req := httptest.NewRequest("POST", "/api/v1/users", bytes.NewBuffer(jsonBody))
    req.Header.Set("Content-Type", "application/json")
    resp, err := app.Test(req)

    require.NoError(t, err)
    assert.Equal(t, 201, resp.StatusCode)

    // Parse response
    var createResp domain.User
    err = json.NewDecoder(resp.Body).Decode(&createResp)
    require.NoError(t, err)
    assert.Equal(t, "integration@test.com", createResp.Email)

    // Get user
    req = httptest.NewRequest("GET", "/api/v1/users/integration-123", nil)
    resp, err = app.Test(req)

    require.NoError(t, err)
    assert.Equal(t, 200, resp.StatusCode)

    var getResp domain.User
    err = json.NewDecoder(resp.Body).Decode(&getResp)
    require.NoError(t, err)
    assert.Equal(t, "integration-123", getResp.ID)
    assert.Equal(t, "integration@test.com", getResp.Email)
    
    // Verify all mock expectations were met
    mockRepo.AssertExpectations(t)
}
```

## Makefile

```makefile
.PHONY: test test-unit test-integration test-coverage

test:
	go test -v ./...

test-unit:
	go test -v ./internal/...

test-integration:
	go test -v ./test/...

test-coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

test-watch:
	find . -name "*.go" | entr -r go test ./...

mock-gen:
	mockery --all --output=test/mocks

clean:
	rm -f coverage.out coverage.html
```

## สรุป

ตัวอย่างนี้แสดง Clean Architecture ที่เรียบง่ายและใช้งานได้จริง:

1. **Testify** - assert, require, mock สำหรับ testing ที่สะดวก
2. **SQLMock** - mock database queries โดยไม่ต้องใช้ database จริง  
3. **Go-Fiber** - fast HTTP framework แทน net/http
4. **Single Repository** - ใช้ PostgreSQL repository เดียว พร้อม mock สำหรับ testing
5. **Clean separation** - แยก domain, usecase, repository, handler ชัดเจน

**ข้อดีของการใช้ Repository เดียว:**
- ✅ ลดความซับซ้อน - ไม่ต้องดูแล memory repository
- ✅ ลด code duplication - implementation เดียว
- ✅ ใช้ SQLMock สำหรับ unit testing
- ✅ ใช้ mock repository สำหรับ integration testing
- ✅ เขียน test ง่ายและรวดเร็ว
- ✅ Maintainable และ scalable

**Perfect สำหรับ:**
- โปรเจกต์ที่ต้องการความเรียบง่าย
- ทีมที่เริ่มต้นใช้ Clean Architecture
- Application ที่ใช้ database หลักเป็น PostgreSQL