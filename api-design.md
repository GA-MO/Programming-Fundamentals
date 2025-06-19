# API Design - การออกแบบ API ที่ดีและใช้งานง่าย

## 📋 สารบัญ
- [API คืออะไร?](#api-คืออะไร)
- [หลักการออกแบบ API ที่ดี](#หลักการออกแบบ-api-ที่ดี)
- [RESTful API Design](#restful-api-design)
- [การตั้งชื่อ Endpoints](#การตั้งชื่อ-endpoints)
- [HTTP Status Codes](#http-status-codes)
- [การจัดการ Errors](#การจัดการ-errors)
- [API Versioning](#api-versioning)
- [Documentation](#documentation)

## API คืออะไร?

**API (Application Programming Interface)** คือตัวกลางที่ให้แอปพลิเคชันต่างๆ สื่อสารกันได้

### ตัวอย่างการใช้งาน API
```
ผู้ใช้ → Mobile App → API → Database
                    ↓
                 Response
```

## หลักการออกแบบ API ที่ดี

### 1. **Consistency (ความสม่ำเสมอ)**
```bash
# ดี - ใช้รูปแบบเดียวกัน
GET /users
GET /products
GET /orders

# ไม่ดี - ไม่สม่ำเสมอ
GET /users
GET /getAllProducts
GET /order-list
```

### 2. **Intuitive (เข้าใจง่าย)**
```bash
# ดี - เข้าใจได้ทันที
GET /users/123/orders
POST /users/123/orders

# ไม่ดี - งงว่าทำอะไร
GET /users/123/data
POST /users/123/action
```

### 3. **Predictable (คาดเดาได้)**
```bash
# ถ้ามี GET /users/123
# ก็ควรมี PUT /users/123, DELETE /users/123
```

## RESTful API Design

### HTTP Methods และการใช้งาน

| Method | ใช้สำหรับ | ตัวอย่าง |
|--------|-----------|----------|
| `GET` | ดึงข้อมูล | `GET /users` - ดูรายการผู้ใช้ |
| `POST` | สร้างข้อมูลใหม่ | `POST /users` - สร้างผู้ใช้ใหม่ |
| `PUT` | อัพเดททั้งหมด | `PUT /users/123` - อัพเดทผู้ใช้ |
| `PATCH` | อัพเดทบางส่วน | `PATCH /users/123` - อัพเดทบางฟิลด์ |
| `DELETE` | ลบข้อมูล | `DELETE /users/123` - ลบผู้ใช้ |

### ตัวอย่าง RESTful API
```bash
# Users
GET    /users           # ดูรายการผู้ใช้ทั้งหมด
GET    /users/123       # ดูข้อมูลผู้ใช้ ID 123
POST   /users           # สร้างผู้ใช้ใหม่
PUT    /users/123       # อัพเดทผู้ใช้ ID 123
DELETE /users/123       # ลบผู้ใช้ ID 123

# Orders ของ User
GET    /users/123/orders     # ดูออเดอร์ของผู้ใช้ 123
POST   /users/123/orders     # สร้างออเดอร์ใหม่สำหรับผู้ใช้ 123
```

## การตั้งชื่อ Endpoints

### ✅ แนวทางที่ดี
```bash
# ใช้คำนาม พหูพจน์
GET /users
GET /products
GET /orders

# ใช้ hyphen สำหรับ compound words
GET /user-profiles
GET /order-items

# Nested resources
GET /users/123/orders
GET /orders/456/items
```

### ❌ แนวทางที่ไม่ดี
```bash
# ไม่ใช้กริยา
GET /getUsers
POST /createUser
DELETE /deleteUser

# ไม่ใช้ camelCase
GET /userProfiles
GET /orderItems

# ไม่ซ้อนลึกเกินไป
GET /users/123/orders/456/items/789/details
```

## HTTP Status Codes

### Status Codes ที่ใช้บ่อย

| Code | ความหมาย | เมื่อไหร่ใช้ |
|------|-----------|-------------|
| `200` | OK | GET สำเร็จ |
| `201` | Created | POST สร้างข้อมูลสำเร็จ |
| `204` | No Content | DELETE สำเร็จ |
| `400` | Bad Request | ข้อมูลที่ส่งมาผิด |
| `401` | Unauthorized | ไม่ได้ login |
| `403` | Forbidden | ไม่มีสิทธิ์ |
| `404` | Not Found | หาข้อมูลไม่เจอ |
| `500` | Internal Server Error | เซิร์ฟเวอร์เกิดข้อผิดพลาด |

### ตัวอย่างการใช้งาน
```javascript
// ดึงข้อมูลสำเร็จ
HTTP 200 OK
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com"
}

// สร้างข้อมูลสำเร็จ
HTTP 201 Created
{
  "id": 124,
  "name": "Jane Doe",
  "email": "jane@example.com"
}

// หาข้อมูลไม่เจอ
HTTP 404 Not Found
{
  "error": "User not found",
  "message": "User with ID 999 does not exist"
}
```

## การจัดการ Errors

### Error Response Structure
```javascript
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Email is required"
      },
      {
        "field": "age",
        "message": "Age must be greater than 0"
      }
    ]
  }
}
```

### ตัวอย่าง Error Responses
```javascript
// 400 Bad Request
{
  "error": {
    "code": "INVALID_EMAIL",
    "message": "Email format is invalid"
  }
}

// 401 Unauthorized
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Authentication required"
  }
}

// 403 Forbidden
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "You don't have permission to access this resource"
  }
}
```

## API Versioning

### 1. URL Versioning (แนะนำ)
```bash
# Version ใน URL path
GET /v1/users
GET /v2/users

# Version ใน subdomain
GET https://api-v1.example.com/users
GET https://api-v2.example.com/users
```

### 2. Header Versioning
```bash
GET /users
Headers:
  API-Version: v1
  Accept: application/vnd.api+json;version=1
```

### 3. Query Parameter Versioning
```bash
GET /users?version=v1
GET /users?api_version=2
```

## Documentation

### สิ่งที่ Documentation ควรมี

#### 1. **Overview**
- API ทำอะไร
- Base URL
- Authentication

#### 2. **Endpoints**
```markdown
### GET /users

**Description:** ดึงรายการผู้ใช้ทั้งหมด

**Parameters:**
- `page` (optional): หน้าที่ต้องการ (default: 1)
- `limit` (optional): จำนวนต่อหน้า (default: 10)

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100
  }
}
```

**Status Codes:**
- 200: Success
- 500: Internal Server Error
```

#### 3. **Examples**
```bash
# Request
curl -X GET "https://api.example.com/v1/users?page=1&limit=5" \
  -H "Authorization: Bearer your-token"

# Response
{
  "data": [...],
  "pagination": {...}
}
```

## 🎯 Best Practices

### 1. **Use HTTPS**
```bash
# ดี
https://api.example.com/users

# ไม่ดี
http://api.example.com/users
```

### 2. **Pagination สำหรับ List APIs**
```javascript
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "has_next": true,
    "has_prev": false
  }
}
```

### 3. **Rate Limiting**
```bash
# Response Headers
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640995200
```

### 4. **Request/Response Logging**
```bash
# Log ทุก request/response เพื่อ debug
[2024-01-01 10:00:00] GET /users/123 - 200 - 45ms
[2024-01-01 10:00:01] POST /users - 400 - 12ms
```

## 🚫 สิ่งที่ควรหลีกเลี่ยง

- ❌ ใช้กริยาใน URL: `/getUsers`, `/createUser`
- ❌ ไม่ consistent: `/users` vs `/user_list`
- ❌ ไม่มี error handling ที่ดี
- ❌ ไม่มี API documentation
- ❌ Response ที่ไม่ consistent structure
- ❌ ไม่ใช้ HTTP status codes อย่างถูกต้อง

---

## 📚 แนวทางการเรียนรู้ต่อ

1. ✅ ฝึกออกแบบ API สำหรับโปรเจ็กต์เล็กๆ
2. ✅ ศึกษา API ของบริษัทใหญ่ เช่น GitHub API, Twitter API
3. ✅ ใช้เครื่องมือ เช่น Postman, Swagger สำหรับทดสอบ
4. ✅ เรียนรู้ GraphQL เป็นทางเลือกของ REST
5. ✅ ศึกษา API Security เพิ่มเติม 