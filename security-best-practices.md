# Security Best Practices - หลักการรักษาความปลอดภัยในการพัฒนา

## 📋 สารบัญ
- [Security คืออะไร?](#security-คืออะไร)
- [Authentication & Authorization](#authentication--authorization)
- [Input Validation & Sanitization](#input-validation--sanitization)
- [HTTPS & Encryption](#https--encryption)
- [Common Vulnerabilities](#common-vulnerabilities)
- [API Security](#api-security)
- [Database Security](#database-security)
- [Frontend Security](#frontend-security)

## Security คืออะไร?

**Security** คือการปกป้องระบบ ข้อมูล และผู้ใช้จากภัยคุกคามต่างๆ เป็นหนึ่งในสิ่งสำคัญที่สุดในการพัฒนาซอฟต์แวร์

### ทำไม Security ถึงสำคัญ?
- 🔒 **ปกป้องข้อมูลผู้ใช้**: ข้อมูลส่วนตัว การเงิน
- 💼 **ปกป้องธุรกิจ**: ชื่อเสียง ความเชื่อถือ
- ⚖️ **ปฏิบัติตามกฎหมาย**: GDPR, PDPA
- 💰 **ประหยัดต้นทุน**: แก้ไขหลังถูกโจมตีแพงกว่า

### CIA Triad - หลักการรักษาความปลอดภัยพื้นฐาน

| หลักการ | คำอธิบาย | ตัวอย่าง |
|---------|-----------|----------|
| **Confidentiality** | ข้อมูลลับต้องเข้าถึงได้เฉพาะคนที่ได้รับอนุญาต | เข้ารหัสรหัสผ่าน |
| **Integrity** | ข้อมูลต้องถูกต้อง ไม่ถูกแก้ไข | Digital signature |
| **Availability** | ระบบต้องพร้อมใช้งานเมื่อต้องการ | DDoS protection |

## Authentication & Authorization

### 1. **Password Security**

#### Password Hashing
```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "golang.org/x/crypto/bcrypt"
    "github.com/gin-gonic/gin"
)

// ❌ ห้ามเก็บรหัสผ่านแบบ plain text
type User struct {
    Email    string `json:"email"`
    Password string `json:"password"` // อันตราย!
}

// ✅ ใช้ bcrypt สำหรับ hash password
const (
    DefaultSaltRounds = 12 // ยิ่งสูงยิ่งปลอดภัย แต่ช้า
    MinPasswordLength = 8  // ความยาวรหัสผ่านขั้นต่ำ
)

const SaltRounds = DefaultSaltRounds

func HashPassword(password string) (string, error) {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), SaltRounds)
    if err != nil {
        return "", err
    }
    return string(hashedPassword), nil
}

func VerifyPassword(password, hashedPassword string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hashedPassword), []byte(password))
    return err == nil
}

// การใช้งาน
type RegisterRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
}

func RegisterHandler(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // ตรวจสอบความแข็งแรงของรหัสผ่าน
    if len(req.Password) < MinPasswordLength {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Password too short"})
        return
    }
    
    hashedPassword, err := HashPassword(req.Password)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to hash password"})
        return
    }
    
    // สร้าง user ในฐานข้อมูล
    user := User{Email: req.Email, Password: hashedPassword}
    // err = db.Create(&user) // ใช้ GORM
    
    c.JSON(http.StatusOK, gin.H{"message": "User created successfully"})
}
```

#### Password Policy
```go
import (
    "regexp"
    "strings"
)

const (
    DefaultMinLength     = 8
    DefaultMaxAge        = 90 // วัน  
    DefaultPreventReuse  = 5  // รหัสผ่านเก่า 5 ตัวล่าสุด
)

type PasswordPolicy struct {
    MinLength        int
    RequireUppercase bool
    RequireLowercase bool
    RequireNumbers   bool
    RequireSymbols   bool
    MaxAge          int // วัน
    PreventReuse    int // รหัสผ่านเก่า 5 ตัวล่าสุด
}

var DefaultPasswordPolicy = PasswordPolicy{
    MinLength:        DefaultMinLength,
    RequireUppercase: true,
    RequireLowercase: true,
    RequireNumbers:   true,
    RequireSymbols:   true,
    MaxAge:          DefaultMaxAge,
    PreventReuse:    DefaultPreventReuse,
}

// ✅ Clean Code: แยกการตรวจสอบออกเป็น functions ย่อย
func ValidatePassword(password string, policy PasswordPolicy) []string {
    var errors []string
    
    if err := validatePasswordLength(password, policy.MinLength); err != nil {
        errors = append(errors, err.Error())
    }
    
    if policy.RequireUppercase {
        if err := validateUppercase(password); err != nil {
            errors = append(errors, err.Error())
        }
    }
    
    if policy.RequireLowercase {
        if err := validateLowercase(password); err != nil {
            errors = append(errors, err.Error())
        }
    }
    
    if policy.RequireNumbers {
        if err := validateNumbers(password); err != nil {
            errors = append(errors, err.Error())
        }
    }
    
    if policy.RequireSymbols {
        if err := validateSymbols(password); err != nil {
            errors = append(errors, err.Error())
        }
    }
    
    return errors
}

func validatePasswordLength(password string, minLength int) error {
    if len(password) < minLength {
        return fmt.Errorf("password must be at least %d characters long", minLength)
    }
    return nil
}

func validateUppercase(password string) error {
    if matched, _ := regexp.MatchString(`[A-Z]`, password); !matched {
        return errors.New("password must contain at least one uppercase letter")
    }
    return nil
}

func validateLowercase(password string) error {
    if matched, _ := regexp.MatchString(`[a-z]`, password); !matched {
        return errors.New("password must contain at least one lowercase letter")
    }
    return nil
}

func validateNumbers(password string) error {
    if matched, _ := regexp.MatchString(`\d`, password); !matched {
        return errors.New("password must contain at least one number")
    }
    return nil
}

func validateSymbols(password string) error {
    if matched, _ := regexp.MatchString(`[!@#$%^&*]`, password); !matched {
        return errors.New("password must contain at least one symbol (!@#$%^&*)")
    }
    return nil
}
```

### 2. **JWT (JSON Web Tokens)**

#### JWT Implementation
```go
import (
    "errors"
    "os"
    "strings"
    "time"
    "github.com/golang-jwt/jwt/v5"
    "github.com/gin-gonic/gin"
)

const (
    AccessTokenDuration = 1 * time.Hour     // Access token หมดอายุใน 1 ชั่วโมง
    TokenExpirationTime = int(AccessTokenDuration.Seconds())
)

var JWT_SECRET = []byte(os.Getenv("JWT_SECRET")) // ต้องเป็น random string ยาวๆ

type Claims struct {
    UserID uint   `json:"userId"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

type UserInfo struct {
    ID    uint   `json:"id"`
    Email string `json:"email"`
    Role  string `json:"role"`
}

func GenerateToken(user UserInfo) (string, error) {
    claims := Claims{
        UserID: user.ID,
        Email:  user.Email,
        Role:   user.Role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(AccessTokenDuration)), // Token หมดอายุใน 1 ชั่วโมง
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",                                            // ระบุผู้ออก token
            Audience:  []string{"myapp-users"},                            // ระบุผู้ใช้ token
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(JWT_SECRET)
}

func VerifyToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return JWT_SECRET, nil
    })
    
    if err != nil {
        if errors.Is(err, jwt.ErrTokenExpired) {
            return nil, errors.New("token expired")
        }
        return nil, errors.New("invalid token")
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, errors.New("invalid token")
}

// Middleware สำหรับตรวจสอบ authentication
func AuthenticateToken() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Access token required"})
            c.Abort()
            return
        }
        
        // Bearer TOKEN
        tokenParts := strings.Split(authHeader, " ")
        if len(tokenParts) != 2 || tokenParts[0] != "Bearer" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid authorization format"})
            c.Abort()
            return
        }
        
        claims, err := VerifyToken(tokenParts[1])
        if err != nil {
            c.JSON(http.StatusForbidden, gin.H{"error": err.Error()})
            c.Abort()
            return
        }
        
        c.Set("user", claims)
        c.Next()
    }
}
```

#### Refresh Token Pattern
```go
import (
    "gorm.io/gorm"
)

const (
    RefreshTokenDuration = 7 * 24 * time.Hour // Refresh token หมดอายุใน 7 วัน
)

var (
    REFRESH_TOKEN_SECRET  = []byte(os.Getenv("REFRESH_TOKEN_SECRET"))
    ErrInvalidCredentials = errors.New("invalid email or password")
    ErrUserNotFound      = errors.New("user not found")
    ErrTokenExpired      = errors.New("token has expired")
)

type RefreshClaims struct {
    UserID uint `json:"userId"`
    jwt.RegisteredClaims
}

// Refresh token ที่มีอายุยาวกว่า
func GenerateRefreshToken(user UserInfo) (string, error) {
    claims := RefreshClaims{
        UserID: user.ID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(RefreshTokenDuration)), // 7 วัน
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(REFRESH_TOKEN_SECRET)
}

type LoginRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required"`
}

type LoginResponse struct {
    AccessToken  string `json:"accessToken"`
    RefreshToken string `json:"refreshToken"`
    ExpiresIn    int    `json:"expiresIn"`
}

type RefreshTokenRequest struct {
    RefreshToken string `json:"refreshToken" binding:"required"`
}

func LoginHandler(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // ค้นหา user ในฐานข้อมูล
    var user User
    if err := db.Where("email = ?", req.Email).First(&user).Error; err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }
    
    if !VerifyPassword(req.Password, user.Password) {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }
    
    userInfo := UserInfo{ID: user.ID, Email: user.Email, Role: user.Role}
    
    accessToken, err := GenerateToken(userInfo)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate access token"})
        return
    }
    
    refreshToken, err := GenerateRefreshToken(userInfo)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate refresh token"})
        return
    }
    
    // เก็บ refresh token ในฐานข้อมูล
    user.RefreshToken = refreshToken
    db.Save(&user)
    
    c.JSON(http.StatusOK, LoginResponse{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
        ExpiresIn:    TokenExpirationTime,
    })
}

func RefreshTokenHandler(c *gin.Context) {
    var req RefreshTokenRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    token, err := jwt.ParseWithClaims(req.RefreshToken, &RefreshClaims{}, func(token *jwt.Token) (interface{}, error) {
        return REFRESH_TOKEN_SECRET, nil
    })
    
    if err != nil {
        c.JSON(http.StatusForbidden, gin.H{"error": "Invalid refresh token"})
        return
    }
    
    if claims, ok := token.Claims.(*RefreshClaims); ok && token.Valid {
        var user User
        if err := db.First(&user, claims.UserID).Error; err != nil {
            c.JSON(http.StatusForbidden, gin.H{"error": "Invalid refresh token"})
            return
        }
        
        if user.RefreshToken != req.RefreshToken {
            c.JSON(http.StatusForbidden, gin.H{"error": "Invalid refresh token"})
            return
        }
        
        userInfo := UserInfo{ID: user.ID, Email: user.Email, Role: user.Role}
        newAccessToken, err := GenerateToken(userInfo)
        if err != nil {
            c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate token"})
            return
        }
        
        c.JSON(http.StatusOK, gin.H{"accessToken": newAccessToken})
    } else {
        c.JSON(http.StatusForbidden, gin.H{"error": "Invalid refresh token"})
    }
}
```

### 3. **Multi-Factor Authentication (MFA)**

#### TOTP (Time-based One-Time Password)
```go
import (
    "crypto/rand"
    "encoding/base32"
    "fmt"
    "github.com/pquerna/otp"
    "github.com/pquerna/otp/totp"
    "github.com/skip2/go-qrcode"
    "strconv"
    "time"
)

const (
    SecretKeyLength = 20  // ความยาว secret key สำหรับ 2FA
    QRCodeSize      = 256 // ขนาด QR code ในพิกเซล
)

type Setup2FAResponse struct {
    Secret string `json:"secret"`
    QRCode string `json:"qrCode"`
}

type Verify2FARequest struct {
    Token string `json:"token" binding:"required"`
}

type Login2FARequest struct {
    Email           string `json:"email" binding:"required,email"`
    Password        string `json:"password" binding:"required"`
    TwoFactorToken  string `json:"twoFactorToken"`
}

// สร้าง secret สำหรับ user
func Setup2FAHandler(c *gin.Context) {
    claims, _ := c.Get("user")
    userClaims := claims.(*Claims)
    
    // สร้าง secret key
    secret := make([]byte, SecretKeyLength)
    rand.Read(secret)
    secretBase32 := base32.StdEncoding.EncodeToString(secret)
    
    // สร้าง OTP URL
    issuer := "MyApp"
    accountName := fmt.Sprintf("MyApp (%s)", userClaims.Email)
    url := fmt.Sprintf("otpauth://totp/%s?secret=%s&issuer=%s", 
        accountName, secretBase32, issuer)
    
    // เก็บ secret ในฐานข้อมูล (ยังไม่ enable)
    var user User
    db.First(&user, userClaims.UserID)
    user.TwoFactorSecret = secretBase32
    user.TwoFactorEnabled = false
    db.Save(&user)
    
    // สร้าง QR code สำหรับ Google Authenticator
    qrCode, err := qrcode.New(url, qrcode.Medium)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate QR code"})
        return
    }
    
    qrCodePNG, err := qrCode.PNG(QRCodeSize)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate QR code"})
        return
    }
    
    c.JSON(http.StatusOK, Setup2FAResponse{
        Secret: secretBase32,
        QRCode: fmt.Sprintf("data:image/png;base64,%s", qrCodePNG),
    })
}

// ยืนยันและเปิดใช้งาน 2FA
func Verify2FAHandler(c *gin.Context) {
    var req Verify2FARequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    claims, _ := c.Get("user")
    userClaims := claims.(*Claims)
    
    var user User
    db.First(&user, userClaims.UserID)
    
    // ตรวจสอบ TOTP token
    valid := totp.Validate(req.Token, user.TwoFactorSecret)
    
    if valid {
        user.TwoFactorEnabled = true
        db.Save(&user)
        c.JSON(http.StatusOK, gin.H{"message": "2FA enabled successfully"})
    } else {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid 2FA token"})
    }
}

// Login ด้วย 2FA
func Login2FAHandler(c *gin.Context) {
    var req Login2FARequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    var user User
    if err := db.Where("email = ?", req.Email).First(&user).Error; err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }
    
    if !VerifyPassword(req.Password, user.Password) {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }
    
    if user.TwoFactorEnabled {
        if req.TwoFactorToken == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "2FA token required"})
            return
        }
        
        valid := totp.Validate(req.TwoFactorToken, user.TwoFactorSecret)
        if !valid {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid 2FA token"})
            return
        }
    }
    
    userInfo := UserInfo{ID: user.ID, Email: user.Email, Role: user.Role}
    
    accessToken, err := GenerateToken(userInfo)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate access token"})
        return
    }
    
    refreshToken, err := GenerateRefreshToken(userInfo)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate refresh token"})
        return
    }
    
    // เก็บ refresh token ในฐานข้อมูล
    user.RefreshToken = refreshToken
    db.Save(&user)
    
    c.JSON(http.StatusOK, LoginResponse{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
        ExpiresIn:    TokenExpirationTime,
    })
}
```

## Input Validation & Sanitization

### 1. **Input Validation**

#### Using go-playground/validator for Validation
```go
import (
    "github.com/go-playground/validator/v10"
    "github.com/gin-gonic/gin"
    "regexp"
    "strings"
)

const (
    MaxEmailLength    = 255
    MaxPasswordLength = 128
    MaxNameLength     = 50
    MinNameLength     = 2
    MinAge            = 13
    MaxAge            = 120
)

var validate = validator.New()

// Custom validation function for strong password
func validateStrongPassword(fl validator.FieldLevel) bool {
    password := fl.Field().String()
    
    // Check for uppercase, lowercase, number and symbol
    hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
    hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
    hasNumber := regexp.MustCompile(`\d`).MatchString(password)
    hasSymbol := regexp.MustCompile(`[!@#$%^&*]`).MatchString(password)
    
    return hasUpper && hasLower && hasNumber && hasSymbol
}

// Register custom validation
func init() {
    validate.RegisterValidation("strongpassword", validateStrongPassword)
}

// Struct สำหรับ user registration
// Note: Go ไม่สามารถใช้ constants ใน struct tags ได้โดยตรง
type UserRegistrationRequest struct {
    Email     string `json:"email" validate:"required,email,max=255"`       // max=MaxEmailLength
    Password  string `json:"password" validate:"required,min=8,max=128,strongpassword"` // min=MinPasswordLength, max=MaxPasswordLength
    FirstName string `json:"firstName" validate:"required,alphanum,min=2,max=50"`        // min=MinNameLength, max=MaxNameLength
    LastName  string `json:"lastName" validate:"required,alphanum,min=2,max=50"`         // min=MinNameLength, max=MaxNameLength
    Age       *int   `json:"age" validate:"omitempty,min=13,max=120"`       // min=MinAge, max=MaxAge
}

type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

// Validation middleware
func ValidateInput() gin.HandlerFunc {
    return func(c *gin.Context) {
        // ใช้ ShouldBindJSON จะ validate อัตโนมัติ
        c.Next()
    }
}

func getValidationErrors(err error) []ValidationError {
    var validationErrors []ValidationError
    
    if validationErr, ok := err.(validator.ValidationErrors); ok {
        for _, fieldError := range validationErr {
            var message string
            
            switch fieldError.Tag() {
            case "required":
                message = "This field is required"
            case "email":
                message = "Must be a valid email address"
            case "min":
                message = fmt.Sprintf("Must be at least %s characters", fieldError.Param())
            case "max":
                message = fmt.Sprintf("Must be at most %s characters", fieldError.Param())
            case "strongpassword":
                message = "Password must contain uppercase, lowercase, number and symbol"
            case "alphanum":
                message = "Must contain only alphanumeric characters"
            default:
                message = "Invalid value"
            }
            
            validationErrors = append(validationErrors, ValidationError{
                Field:   strings.ToLower(fieldError.Field()),
                Message: message,
            })
        }
    }
    
    return validationErrors
}

// ใช้งาน
func UserRegistrationHandler(c *gin.Context) {
    var req UserRegistrationRequest
    
    if err := c.ShouldBindJSON(&req); err != nil {
        // Handle binding errors
        if validationErrors := getValidationErrors(err); len(validationErrors) > 0 {
            c.JSON(http.StatusBadRequest, gin.H{
                "error":   "Validation failed",
                "details": validationErrors,
            })
            return
        }
        
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid JSON"})
        return
    }
    
    // Additional validation using validator
    if err := validate.Struct(&req); err != nil {
        validationErrors := getValidationErrors(err)
        c.JSON(http.StatusBadRequest, gin.H{
            "error":   "Validation failed",
            "details": validationErrors,
        })
        return
    }
    
    // ข้อมูลได้ผ่านการ validate แล้ว
    hashedPassword, err := HashPassword(req.Password)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to process request"})
        return
    }
    
    user := User{
        Email:     req.Email,
        Password:  hashedPassword,
        FirstName: req.FirstName,
        LastName:  req.LastName,
    }
    
    if req.Age != nil {
        user.Age = *req.Age
    }
    
    // บันทึกลงฐานข้อมูล
    if err := db.Create(&user).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to create user"})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{"message": "User created successfully"})
}
```

### 2. **SQL Injection Prevention**

#### Parameterized Queries
```go
import (
    "database/sql"
    "strconv"
    "gorm.io/gorm"
    _ "github.com/go-sql-driver/mysql"
)

// ❌ SQL Injection vulnerability
func GetUserUnsafe(c *gin.Context) {
    userID := c.Param("id")
    // อันตราย! user สามารถใส่: 1 OR 1=1
    query := fmt.Sprintf("SELECT * FROM users WHERE id = %s", userID)
    
    rows, err := db.Query(query)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Database error"})
        return
    }
    defer rows.Close()
    
    // Process results...
}

// ✅ ใช้ parameterized queries
func GetUserSafe(c *gin.Context) {
    userIDStr := c.Param("id")
    userID, err := strconv.Atoi(userIDStr)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid user ID"})
        return
    }
    
    query := "SELECT id, email, first_name, last_name FROM users WHERE id = ?"
    
    var user User
    err = db.QueryRow(query, userID).Scan(&user.ID, &user.Email, &user.FirstName, &user.LastName)
    if err != nil {
        if err == sql.ErrNoRows {
            c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
            return
        }
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Database error"})
        return
    }
    
    c.JSON(http.StatusOK, user)
}

// ✅ ใช้ GORM (ORM)
func GetUserWithGORM(c *gin.Context) {
    userIDStr := c.Param("id")
    userID, err := strconv.Atoi(userIDStr)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid user ID"})
        return
    }
    
    var user User
    // GORM handle parameterization automatically
    if err := db.First(&user, userID).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
            return
        }
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Database error"})
        return
    }
    
    c.JSON(http.StatusOK, user)
}

// ✅ Complex query with GORM
func SearchUsers(c *gin.Context) {
    name := c.Query("name")
    email := c.Query("email")
    
    var users []User
    query := db.Model(&User{})
    
    if name != "" {
        // GORM จะ escape อัตโนมัติ
        query = query.Where("first_name LIKE ? OR last_name LIKE ?", "%"+name+"%", "%"+name+"%")
    }
    
    if email != "" {
        query = query.Where("email LIKE ?", "%"+email+"%")
    }
    
    if err := query.Find(&users).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Database error"})
        return
    }
    
    c.JSON(http.StatusOK, users)
}
```

### 3. **XSS Prevention**

#### Output Encoding
```go
import (
    "html"
    "regexp"
    "strings"
    "time"
)

// HTML Sanitizer
type HTMLSanitizer struct {
    allowedTags map[string]bool
    allowedAttrs map[string]bool
}

func NewHTMLSanitizer() *HTMLSanitizer {
    return &HTMLSanitizer{
        allowedTags: map[string]bool{
            "p": true, "br": true, "strong": true, "em": true,
            "ul": true, "ol": true, "li": true,
        },
        allowedAttrs: map[string]bool{},
    }
}

// ✅ Clean Code: แยกการทำงานออกเป็น methods ย่อย
func (s *HTMLSanitizer) Sanitize(input string) string {
    cleaned := input
    cleaned = s.removeScriptTags(cleaned)
    cleaned = s.removeJavaScriptURLs(cleaned)
    cleaned = s.removeEventHandlers(cleaned)
    cleaned = s.escapeHTMLEntities(cleaned)
    return cleaned
}

func (s *HTMLSanitizer) removeScriptTags(input string) string {
    scriptRegex := regexp.MustCompile(`<script[\s\S]*?</script>`)
    return scriptRegex.ReplaceAllString(input, "")
}

func (s *HTMLSanitizer) removeJavaScriptURLs(input string) string {
    jsRegex := regexp.MustCompile(`javascript:`)
    return jsRegex.ReplaceAllString(input, "")
}

func (s *HTMLSanitizer) removeEventHandlers(input string) string {
    onEventRegex := regexp.MustCompile(`on\w+\s*=\s*"[^"]*"`)
    return onEventRegex.ReplaceAllString(input, "")
}

func (s *HTMLSanitizer) escapeHTMLEntities(input string) string {
    return html.EscapeString(input)
}

// Simple text sanitizer
func SanitizeText(input string) string {
    // Remove HTML tags completely
    htmlRegex := regexp.MustCompile(`<[^>]*>`)
    clean := htmlRegex.ReplaceAllString(input, "")
    
    // Escape remaining special characters
    clean = html.EscapeString(clean)
    
    return strings.TrimSpace(clean)
}

type CommentRequest struct {
    Content string `json:"content" binding:"required"`
}

type Comment struct {
    ID        uint      `json:"id"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"createdAt"`
}

// ❌ XSS vulnerability
func CreateCommentUnsafe(c *gin.Context) {
    var req CommentRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // ถ้า content = "<script>alert('XSS')</script>"
    comment := Comment{
        Content:   req.Content, // อันตราย!
        CreatedAt: time.Now(),
    }
    
    c.JSON(http.StatusOK, comment)
}

// ✅ Sanitize input
func CreateCommentSafe(c *gin.Context) {
    var req CommentRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    sanitizer := NewHTMLSanitizer()
    comment := Comment{
        Content:   sanitizer.Sanitize(req.Content), // ปลอดภัย
        CreatedAt: time.Now(),
    }
    
    // บันทึกลงฐานข้อมูล
    if err := db.Create(&comment).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to create comment"})
        return
    }
    
    c.JSON(http.StatusOK, comment)
}

// Middleware for automatic XSS protection
func XSSProtectionMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // Set XSS protection headers
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        
        c.Next()
    }
}
```

#### CSP (Content Security Policy)
```javascript
const helmet = require('helmet');

app.use(helmet.contentSecurityPolicy({
    directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        imgSrc: ["'self'", "data:", "https:"],
        scriptSrc: ["'self'"],
        objectSrc: ["'none'"],
        upgradeInsecureRequests: []
    }
}));

// HTML meta tag
// <meta http-equiv="Content-Security-Policy" 
//       content="default-src 'self'; script-src 'self'">
```

## HTTPS & Encryption

### 1. **HTTPS Setup**

#### Gin with HTTPS
```go
import (
    "crypto/tls"
    "fmt"
    "net/http"
    "github.com/gin-gonic/gin"
)

const (
    HTTPSPort       = ":443"
    HTTPPort        = ":80"
    CertFile        = "certificate.pem"
    KeyFile         = "private-key.pem"
    CertsCacheDir   = "certs"
)

func SetupHTTPS() {
    r := gin.Default()
    
    // Middleware to redirect HTTP to HTTPS
    r.Use(func(c *gin.Context) {
        if c.GetHeader("X-Forwarded-Proto") == "http" {
            target := "https://" + c.Request.Host + c.Request.URL.Path
            if c.Request.URL.RawQuery != "" {
                target += "?" + c.Request.URL.RawQuery
            }
            c.Redirect(http.StatusMovedPermanently, target)
            return
        }
        c.Next()
    })
    
    // SSL Certificate
    certFile := CertFile
    keyFile := KeyFile
    
    // TLS configuration
    tlsConfig := &tls.Config{
        MinVersion: tls.VersionTLS12,
        CurvePreferences: []tls.CurveID{
            tls.CurveP521,
            tls.CurveP384,
            tls.CurveP256,
        },
        PreferServerCipherSuites: true,
        CipherSuites: []uint16{
            tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
            tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,
            tls.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
        },
    }
    
    server := &http.Server{
        Addr:      HTTPSPort,
        Handler:   r,
        TLSConfig: tlsConfig,
    }
    
    fmt.Printf("HTTPS Server running on port %s\n", HTTPSPort)
    server.ListenAndServeTLS(certFile, keyFile)
}

// Alternative: Auto HTTPS with Let's Encrypt
func SetupAutoHTTPS() {
    r := gin.Default()
    
    // ใช้ autocert สำหรับ Let's Encrypt
    m := &autocert.Manager{
        Cache:      autocert.DirCache(CertsCacheDir),
        Prompt:     autocert.AcceptTOS,
        HostPolicy: autocert.HostWhitelist("yourdomain.com", "www.yourdomain.com"),
    }
    
    server := &http.Server{
        Addr:      HTTPSPort,
        Handler:   r,
        TLSConfig: m.TLSConfig(),
    }
    
    // HTTP server สำหรับ redirect และ ACME challenge
    go http.ListenAndServe(HTTPPort, m.HTTPHandler(nil))
    
    fmt.Printf("Auto HTTPS Server running on port %s\n", HTTPSPort)
    server.ListenAndServeTLS("", "")
}
```

#### HTTP Security Headers
```javascript
const helmet = require('helmet');

app.use(helmet({
    // HTTPS Strict Transport Security
    hsts: {
        maxAge: 31536000, // 1 year
        includeSubDomains: true,
        preload: true
    },
    
    // Prevent clickjacking
    frameguard: { action: 'deny' },
    
    // Prevent MIME type sniffing
    noSniff: true,
    
    // XSS protection
    xssFilter: true,
    
    // Referrer policy
    referrerPolicy: { policy: 'same-origin' }
}));
```

### 2. **Data Encryption**

#### Symmetric Encryption
```go
import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/hex"
    "encoding/json"
    "errors"
    "io"
)

const (
    AESKeySize = 32 // AES-256 requires 32 bytes key
)

type EncryptionService struct {
    key []byte
}

func NewEncryptionService(key string) *EncryptionService {
    return &EncryptionService{
        key: []byte(key), // Must be AESKeySize bytes for AES-256
    }
}

type EncryptedData struct {
    Encrypted string `json:"encrypted"`
    Nonce     string `json:"nonce"`
}

func (es *EncryptionService) Encrypt(plaintext string) (*EncryptedData, error) {
    // Create AES cipher
    block, err := aes.NewCipher(es.key)
    if err != nil {
        return nil, err
    }
    
    // Create GCM mode
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }
    
    // Generate random nonce
    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }
    
    // Encrypt the data
    ciphertext := gcm.Seal(nil, nonce, []byte(plaintext), nil)
    
    return &EncryptedData{
        Encrypted: hex.EncodeToString(ciphertext),
        Nonce:     hex.EncodeToString(nonce),
    }, nil
}

func (es *EncryptionService) Decrypt(encryptedData *EncryptedData) (string, error) {
    // Create AES cipher
    block, err := aes.NewCipher(es.key)
    if err != nil {
        return "", err
    }
    
    // Create GCM mode
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }
    
    // Decode hex strings
    ciphertext, err := hex.DecodeString(encryptedData.Encrypted)
    if err != nil {
        return "", err
    }
    
    nonce, err := hex.DecodeString(encryptedData.Nonce)
    if err != nil {
        return "", err
    }
    
    // Decrypt the data
    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return "", err
    }
    
    return string(plaintext), nil
}

// Database model with encrypted field
type UserWithEncryption struct {
    ID            uint   `gorm:"primaryKey"`
    Email         string `gorm:"unique;not null"`
    EncryptedData string `gorm:"type:text"`
}

// Helper methods for database operations
func (u *UserWithEncryption) SetSensitiveData(data string, encryption *EncryptionService) error {
    encrypted, err := encryption.Encrypt(data)
    if err != nil {
        return err
    }
    
    jsonData, err := json.Marshal(encrypted)
    if err != nil {
        return err
    }
    
    u.EncryptedData = string(jsonData)
    return nil
}

func (u *UserWithEncryption) GetSensitiveData(encryption *EncryptionService) (string, error) {
    if u.EncryptedData == "" {
        return "", errors.New("no encrypted data found")
    }
    
    var encrypted EncryptedData
    if err := json.Unmarshal([]byte(u.EncryptedData), &encrypted); err != nil {
        return "", err
    }
    
    return encryption.Decrypt(&encrypted)
}

// ใช้งาน
func ExampleUsage() {
    config := LoadConfig()
    encryption := NewEncryptionService(config.EncryptionKey)
    
    // เข้ารหัสข้อมูลส่วนตัว
    sensitiveData := "เลขบัตรประชาชน: 1234567890123"
    
    user := &UserWithEncryption{
        Email: "user@example.com",
    }
    
    // Set encrypted data
    if err := user.SetSensitiveData(sensitiveData, encryption); err != nil {
        log.Printf("Encryption failed: %v", err)
        return
    }
    
    // เก็บในฐานข้อมูล
    if err := db.Create(user).Error; err != nil {
        log.Printf("Database save failed: %v", err)
        return
    }
    
    // อ่านและถอดรหัสข้อมูล
    var retrievedUser UserWithEncryption
    if err := db.First(&retrievedUser, user.ID).Error; err != nil {
        log.Printf("Database retrieve failed: %v", err)
        return
    }
    
    decryptedData, err := retrievedUser.GetSensitiveData(encryption)
    if err != nil {
        log.Printf("Decryption failed: %v", err)
        return
    }
    
    log.Printf("Decrypted data: %s", decryptedData)
}
```

## Common Vulnerabilities

### 1. **OWASP Top 10**

#### A01 - Broken Access Control
```javascript
// ❌ ไม่ตรวจสอบสิทธิ์
app.get('/api/users/:id/profile', (req, res) => {
    const userId = req.params.id;
    // ผู้ใช้สามารถดูข้อมูลคนอื่นได้!
    const profile = getUserProfile(userId);
    res.json(profile);
});

// ✅ ตรวจสอบสิทธิ์
app.get('/api/users/:id/profile', authenticateToken, (req, res) => {
    const requestedUserId = req.params.id;
    const currentUserId = req.user.userId;
    
    // ตรวจสอบว่าเป็นเจ้าของข้อมูลหรือ admin
    if (requestedUserId !== currentUserId && req.user.role !== 'admin') {
        return res.status(403).json({ error: 'Access denied' });
    }
    
    const profile = getUserProfile(requestedUserId);
    res.json(profile);
});
```

#### A03 - Injection
```javascript
// ❌ NoSQL Injection
app.post('/api/login', (req, res) => {
    const { email, password } = req.body;
    // ถ้าส่งมา: { "email": { "$ne": null }, "password": { "$ne": null } }
    const user = db.collection('users').findOne({ email, password });
    // จะ login ได้โดยไม่ต้องรู้รหัสผ่าน!
});

// ✅ Validate input types
app.post('/api/login', (req, res) => {
    const { email, password } = req.body;
    
    // ตรวจสอบว่าเป็น string
    if (typeof email !== 'string' || typeof password !== 'string') {
        return res.status(400).json({ error: 'Invalid input types' });
    }
    
    const user = db.collection('users').findOne({ 
        email: email,
        password: hashPassword(password)
    });
});
```

### 2. **CSRF Prevention**

#### CSRF Token
```javascript
const csrf = require('csurf');

// ตั้งค่า CSRF protection
const csrfProtection = csrf({
    cookie: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict'
    }
});

app.use(csrfProtection);

// ส่ง CSRF token ไปยัง client
app.get('/api/csrf-token', (req, res) => {
    res.json({ csrfToken: req.csrfToken() });
});

// Protected route
app.post('/api/sensitive-action', (req, res) => {
    // CSRF token จะถูกตรวจสอบอัตโนมัติ
    res.json({ message: 'Action completed' });
});
```

#### SameSite Cookies
```javascript
app.use(session({
    secret: process.env.SESSION_SECRET,
    cookie: {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'strict', // ป้องกัน CSRF
        maxAge: 24 * 60 * 60 * 1000 // 24 hours
    }
}));
```

## API Security

### 1. **Rate Limiting**

#### Gin Rate Limit
```go
import (
    "sync"
    "time"
    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

const (
    GeneralRateLimit = 100                // requests per window
    AuthRateLimit    = 5                  // auth attempts per window  
    RateLimitWindow  = 15 * time.Minute   // time window สำหรับ rate limiting
)

// Rate limiter using token bucket algorithm
type RateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
    ttl      time.Duration
}

func NewRateLimiter(rps rate.Limit, burst int, ttl time.Duration) *RateLimiter {
    rl := &RateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     rps,
        burst:    burst,
        ttl:      ttl,
    }
    
    // Cleanup old limiters periodically
    go rl.cleanupLimiters()
    
    return rl
}

func (rl *RateLimiter) getLimiter(key string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    limiter, exists := rl.limiters[key]
    if !exists {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.limiters[key] = limiter
    }
    
    return limiter
}

func (rl *RateLimiter) cleanupLimiters() {
    ticker := time.NewTicker(rl.ttl)
    defer ticker.Stop()
    
    for range ticker.C {
        rl.mu.Lock()
        for key, limiter := range rl.limiters {
            if limiter.TokensAt(time.Now()) == float64(rl.burst) {
                delete(rl.limiters, key)
            }
        }
        rl.mu.Unlock()
    }
}

func (rl *RateLimiter) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        key := c.ClientIP() // ใช้ IP address เป็น key
        limiter := rl.getLimiter(key)
        
        if !limiter.Allow() {
            c.Header("X-RateLimit-Limit", fmt.Sprintf("%.0f", float64(rl.rate)))
            c.Header("X-RateLimit-Remaining", "0")
            c.Header("X-RateLimit-Reset", fmt.Sprintf("%d", time.Now().Add(time.Second/time.Duration(rl.rate)).Unix()))
            
            c.JSON(http.StatusTooManyRequests, gin.H{
                "error": "Too many requests, please try again later",
            })
            c.Abort()
            return
        }
        
        c.Next()
    }
}

// Global rate limiters
var (
    // General rate limiting: 100 requests per 15 minutes
    generalLimiter = NewRateLimiter(
        rate.Limit(GeneralRateLimit)/rate.Limit(RateLimitWindow.Seconds()), 
        GeneralRateLimit, 
        RateLimitWindow,
    )
    
    // Strict rate limiting: 5 requests per 15 minutes
    authLimiter = NewRateLimiter(
        rate.Limit(AuthRateLimit)/rate.Limit(RateLimitWindow.Seconds()), 
        AuthRateLimit, 
        RateLimitWindow,
    )
)

// Advanced rate limiter with user-specific limits
type UserRateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
}

func NewUserRateLimiter() *UserRateLimiter {
    return &UserRateLimiter{
        limiters: make(map[string]*rate.Limiter),
    }
}

func (url *UserRateLimiter) AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        claims, exists := c.Get("user")
        if !exists {
            c.Next()
            return
        }
        
        userClaims := claims.(*Claims)
        key := fmt.Sprintf("user:%d", userClaims.UserID)
        
        url.mu.Lock()
        limiter, exists := url.limiters[key]
        if !exists {
            // Premium users get higher limits
            if userClaims.Role == "premium" {
                limiter = rate.NewLimiter(10, 50) // 10 req/sec, burst 50
            } else {
                limiter = rate.NewLimiter(1, 10)  // 1 req/sec, burst 10
            }
            url.limiters[key] = limiter
        }
        url.mu.Unlock()
        
        if !limiter.Allow() {
            c.JSON(http.StatusTooManyRequests, gin.H{
                "error": "Rate limit exceeded for user",
            })
            c.Abort()
            return
        }
        
        c.Next()
    }
}

// ใช้งาน
func SetupRoutes() {
    r := gin.Default()
    
    // Apply general rate limiting to all API routes
    api := r.Group("/api", generalLimiter.Middleware())
    
    // Apply strict rate limiting to auth routes
    auth := api.Group("/auth", authLimiter.Middleware())
    {
        auth.POST("/login", LoginHandler)
        auth.POST("/register", RegisterHandler)
        auth.POST("/forgot-password", ForgotPasswordHandler)
    }
    
    // User-specific rate limiting for protected routes
    userLimiter := NewUserRateLimiter()
    protected := api.Group("/", AuthenticateToken(), userLimiter.AuthMiddleware())
    {
        protected.GET("/profile", GetProfileHandler)
        protected.POST("/upload", UploadHandler)
    }
}
```

#### Custom Rate Limiting with Redis
```javascript
const redis = require('redis');
const client = redis.createClient();

const rateLimitMiddleware = (maxRequests, windowMs) => {
    return async (req, res, next) => {
        const key = `rate_limit:${req.ip}`;
        
        try {
            const current = await client.incr(key);
            
            if (current === 1) {
                await client.expire(key, Math.ceil(windowMs / 1000));
            }
            
            if (current > maxRequests) {
                return res.status(429).json({
                    error: 'Rate limit exceeded',
                    retryAfter: Math.ceil(windowMs / 1000)
                });
            }
            
            res.set({
                'X-RateLimit-Limit': maxRequests,
                'X-RateLimit-Remaining': Math.max(0, maxRequests - current),
                'X-RateLimit-Reset': Date.now() + windowMs
            });
            
            next();
        } catch (error) {
            next(); // ถ้า Redis error ก็ให้ผ่านไป
        }
    };
};
```

### 2. **API Key Management**

```javascript
// API Key middleware
const validateApiKey = async (req, res, next) => {
    const apiKey = req.headers['x-api-key'];
    
    if (!apiKey) {
        return res.status(401).json({ error: 'API key required' });
    }
    
    try {
        const hashedKey = crypto
            .createHash('sha256')
            .update(apiKey)
            .digest('hex');
            
        const apiKeyRecord = await ApiKey.findOne({ 
            hashedKey,
            isActive: true,
            expiresAt: { $gt: new Date() }
        });
        
        if (!apiKeyRecord) {
            return res.status(401).json({ error: 'Invalid API key' });
        }
        
        // Update last used
        await apiKeyRecord.update({ lastUsedAt: new Date() });
        
        req.apiKey = apiKeyRecord;
        next();
    } catch (error) {
        res.status(500).json({ error: 'Server error' });
    }
};

// สร้าง API key
app.post('/api/api-keys', authenticateToken, async (req, res) => {
    const { name, permissions } = req.body;
    
    const apiKey = crypto.randomBytes(32).toString('hex');
    const hashedKey = crypto
        .createHash('sha256')
        .update(apiKey)
        .digest('hex');
    
    const apiKeyRecord = await ApiKey.create({
        name,
        hashedKey,
        permissions,
        userId: req.user.userId,
        expiresAt: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000) // 1 year
    });
    
    res.json({
        apiKey, // แสดงเฉพาะครั้งเดียว
        id: apiKeyRecord.id,
        name: apiKeyRecord.name
    });
});
```

## Database Security

### 1. **Database Access Control**

```sql
-- สร้าง user เฉพาะสำหรับ application
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'strong_random_password';

-- ให้สิทธิ์เฉพาะที่จำเป็น
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'app_user'@'localhost';

-- ไม่ให้สิทธิ์ admin
-- REVOKE CREATE, DROP, ALTER ON *.* FROM 'app_user'@'localhost';

-- สำหรับ read-only operations
CREATE USER 'readonly_user'@'localhost' IDENTIFIED BY 'another_password';
GRANT SELECT ON myapp.* TO 'readonly_user'@'localhost';
```

### 2. **Database Encryption**

```sql
-- Encrypt sensitive columns
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    -- เข้ารหัสข้อมูลส่วนตัว
    encrypted_ssn VARBINARY(255), -- เลขประชาชน
    encrypted_phone VARBINARY(255), -- เบอร์โทร
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Encrypt table data at rest
CREATE TABLE sensitive_data (
    id INT PRIMARY KEY AUTO_INCREMENT,
    data TEXT
) ENCRYPTION='Y';
```

```javascript
// Application-level encryption
const encryptSensitiveData = (data) => {
    return crypto
        .createCipher('aes-256-cbc', process.env.DB_ENCRYPTION_KEY)
        .update(data, 'utf8', 'hex') + 
        crypto
        .createCipher('aes-256-cbc', process.env.DB_ENCRYPTION_KEY)
        .final('hex');
};

const decryptSensitiveData = (encryptedData) => {
    return crypto
        .createDecipher('aes-256-cbc', process.env.DB_ENCRYPTION_KEY)
        .update(encryptedData, 'hex', 'utf8') +
        crypto
        .createDecipher('aes-256-cbc', process.env.DB_ENCRYPTION_KEY)
        .final('utf8');
};
```

## Frontend Security

### 1. **Secure Local Storage**

```javascript
// ❌ เก็บข้อมูลสำคัญใน localStorage
localStorage.setItem('accessToken', token); // อันตราย!
localStorage.setItem('userPassword', password); // อันตรายมาก!

// ✅ ใช้ httpOnly cookies สำหรับ sensitive data
// Backend ตั้งค่า cookie
res.cookie('accessToken', token, {
    httpOnly: true,    // ไม่สามารถเข้าถึงผ่าน JavaScript
    secure: true,      // HTTPS only
    sameSite: 'strict', // CSRF protection
    maxAge: 3600000    // 1 hour
});

// ✅ หรือใช้ sessionStorage สำหรับข้อมูลไม่สำคัญ
sessionStorage.setItem('userPreferences', JSON.stringify(preferences));

// ✅ Encrypt ก่อนเก็บ (ถ้าจำเป็น)
const encryptedData = CryptoJS.AES.encrypt(
    JSON.stringify(sensitiveData), 
    userSecret
).toString();
localStorage.setItem('encryptedData', encryptedData);
```

### 2. **Input Sanitization (Frontend)**

```javascript
// HTML Sanitization
import DOMPurify from 'dompurify';

const sanitizeHtml = (dirty) => {
    return DOMPurify.sanitize(dirty, {
        ALLOWED_TAGS: ['p', 'br', 'strong', 'em'],
        ALLOWED_ATTR: []
    });
};

// React component
const UserComment = ({ comment }) => {
    const cleanHtml = sanitizeHtml(comment.content);
    
    return (
        <div 
            dangerouslySetInnerHTML={{ __html: cleanHtml }}
        />
    );
};

// Validate user input
const validateEmail = (email) => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
};

const validateInput = (input, type) => {
    switch (type) {
        case 'email':
            return validateEmail(input);
        case 'phone':
            return /^[0-9+\-\s()]+$/.test(input);
        case 'name':
            return /^[a-zA-Z\s]+$/.test(input);
        default:
            return true;
    }
};
```

## 🎯 Security Best Practices

### 1. **Principle of Least Privilege**
```javascript
// ให้สิทธิ์เฉพาะที่จำเป็น
const checkPermission = (user, resource, action) => {
    const userPermissions = user.permissions || [];
    const requiredPermission = `${resource}:${action}`;
    
    return userPermissions.includes(requiredPermission) || 
           userPermissions.includes(`${resource}:*`) ||
           userPermissions.includes('*:*');
};

app.delete('/api/users/:id', authenticateToken, (req, res) => {
    if (!checkPermission(req.user, 'users', 'delete')) {
        return res.status(403).json({ error: 'Insufficient permissions' });
    }
    
    // ดำเนินการลบ
});
```

### 2. **Security Monitoring**
```javascript
const securityLogger = require('./security-logger');

// Log security events
const logSecurityEvent = (event, user, details) => {
    securityLogger.warn({
        event,
        userId: user?.id,
        userAgent: details.userAgent,
        ip: details.ip,
        timestamp: new Date(),
        details
    });
};

// Monitor failed login attempts
app.post('/api/login', async (req, res) => {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    
    if (!user || !await verifyPassword(password, user.password)) {
        logSecurityEvent('FAILED_LOGIN', null, {
            email,
            ip: req.ip,
            userAgent: req.get('User-Agent')
        });
        
        return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    logSecurityEvent('SUCCESSFUL_LOGIN', user, {
        ip: req.ip,
        userAgent: req.get('User-Agent')
    });
    
    // ส่ง token
});
```

### 3. **Environment Variables**
```go
import (
    "fmt"
    "log"
    "os"
)

const (
    MinJWTSecretLength       = 32 // ความยาวขั้นต่ำของ JWT secret
    RequiredEncryptionKeyLen = 32 // ความยาวที่ต้องการสำหรับ encryption key (AES-256)
)

// ❌ Hard-coded secrets
// const JWT_SECRET = "mysecretkey123" // อันตราย!

// ✅ ใช้ environment variables
func getEnvOrFatal(key string) string {
    value := os.Getenv(key)
    if value == "" {
        log.Fatalf("Missing required environment variable: %s", key)
    }
    return value
}

func getEnvOrDefault(key, defaultValue string) string {
    value := os.Getenv(key)
    if value == "" {
        return defaultValue
    }
    return value
}

// Configuration struct
type Config struct {
    Environment    string
    JWTSecret      string
    DBPassword     string
    EncryptionKey  string
    RedisURL       string
    Port           string
    DatabaseURL    string
}

func LoadConfig() *Config {
    return &Config{
        Environment:   getEnvOrDefault("ENV", "development"),
        JWTSecret:     getEnvOrFatal("JWT_SECRET"),
        DBPassword:    getEnvOrFatal("DB_PASSWORD"),
        EncryptionKey: getEnvOrFatal("ENCRYPTION_KEY"),
        RedisURL:      getEnvOrDefault("REDIS_URL", "redis://localhost:6379"),
        Port:          getEnvOrDefault("PORT", "8080"),
        DatabaseURL:   getEnvOrFatal("DATABASE_URL"),
    }
}

// .env file
/*
ENV=production
JWT_SECRET=very_long_random_string_here_minimum_64_characters
DB_PASSWORD=another_secure_password
ENCRYPTION_KEY=32_byte_encryption_key_for_aes_256
DATABASE_URL=postgres://user:password@localhost/myapp?sslmode=require
REDIS_URL=redis://localhost:6379
PORT=8080
*/

// Security validation on startup
func ValidateConfig(config *Config) error {
    // Validate JWT secret length
    if len(config.JWTSecret) < MinJWTSecretLength {
        return fmt.Errorf("JWT_SECRET must be at least %d characters long", MinJWTSecretLength)
    }
    
    // Validate encryption key length (must be 32 bytes for AES-256)
    if len(config.EncryptionKey) != RequiredEncryptionKeyLen {
        return fmt.Errorf("ENCRYPTION_KEY must be exactly %d characters (256 bits)", RequiredEncryptionKeyLen)
    }
    
    // Validate environment
    validEnvs := map[string]bool{
        "development": true,
        "staging":     true,
        "production":  true,
    }
    
    if !validEnvs[config.Environment] {
        return fmt.Errorf("ENV must be one of: development, staging, production")
    }
    
    return nil
}

func main() {
    config := LoadConfig()
    
    if err := ValidateConfig(config); err != nil {
        log.Fatalf("Configuration validation failed: %v", err)
    }
    
    log.Printf("Starting application in %s mode on port %s", 
        config.Environment, config.Port)
    
    // Initialize your application with config
    // app := NewApp(config)
    // app.Run()
}
```

## 🚫 สิ่งที่ควรหลีกเลี่ยง

- ❌ **Hard-code secrets** ในโค้ด
- ❌ **เก็บรหัสผ่านแบบ plain text**
- ❌ **ไม่ validate input**
- ❌ **ใช้ HTTP แทน HTTPS**
- ❌ **ไม่ใช้ rate limiting**
- ❌ **ไม่ log security events**
- ❌ **ให้สิทธิ์มากเกินไป**
- ❌ **ไม่อัพเดท dependencies**

---

## 📚 แนวทางการเรียนรู้ต่อ

1. ✅ ศึกษา OWASP Top 10 อย่างละเอียด
2. ✅ ฝึก Security Testing (Penetration Testing)
3. ✅ เรียนรู้ Cryptography พื้นฐาน
4. ✅ ศึกษา Cloud Security (AWS, GCP, Azure)
5. ✅ เรียนรู้ DevSecOps practices 

 