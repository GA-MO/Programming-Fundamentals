# Git Branching Strategy คู่มือการจัดการ Branch

## Git Branching Strategy คืออะไร?

Git Branching Strategy คือแนวทางการจัดการและตั้งชื่อ branch ใน Git repository ให้เป็นระบบ เพื่อให้ทีมพัฒนาสามารถทำงานร่วมกันได้อย่างมีประสิทธิภาพ ลดความขัดแย้ง และควบคุม code quality ได้ดีขึ้น

## กลยุทธ์ที่นิยมใช้

### 1. Git Flow
กลยุทธ์ที่เหมาะกับโปรเจกต์ที่มี release cycle ชัดเจน

**Branch หลัก:**
- `main/master` - production code
- `develop` - integration branch สำหรับ feature ใหม่
- `feature/*` - พัฒนา feature ใหม่
- `release/*` - เตรียม release
- `hotfix/*` - แก้ไข bug ด่วนใน production

### 2. GitHub Flow
กลยุทธ์ที่เรียบง่าย เหมาะกับ continuous deployment

**Branch หลัก:**
- `main` - production-ready code
- `feature/*` - พัฒนา feature และ bug fix

### 3. GitLab Flow
ผสมผสานจุดเด่นของ Git Flow และ GitHub Flow

## ตัวอย่างที่ดี ✅

### การตั้งชื่อ Branch ที่ดี
```bash
# ชัดเจน บอกประเภทและสิ่งที่ทำ
feature/user-authentication
feature/payment-integration
bugfix/login-error-handling
hotfix/security-vulnerability-patch
release/v2.1.0

# รวม ticket number
feature/PROJ-123-add-search-functionality
bugfix/PROJ-456-fix-memory-leak
```

### Git Flow Workflow ที่ดี
```bash
# 1. สร้าง feature branch จาก develop
git checkout develop
git pull origin develop
git checkout -b feature/user-profile

# 2. พัฒนาและ commit
git add .
git commit -m "feat: add user profile page"

# 3. Push และสร้าง Pull Request
git push origin feature/user-profile

# 4. Merge เข้า develop หลัง code review
# 5. Delete feature branch หลัง merge
git branch -d feature/user-profile
```

### Commit Message ที่ดี
```bash
# ใช้ conventional commits
feat: add user authentication system
fix: resolve memory leak in data processing
docs: update API documentation
test: add unit tests for payment module
refactor: optimize database queries
```

## ตัวอย่างที่ไม่ดี ❌

### การตั้งชื่อ Branch ที่ไม่ดี
```bash
# ไม่ชัดเจน ไม่มีความหมาย
test
temp
fix
new-stuff
john-branch
random-changes
```

### Workflow ที่ไม่ดี
```bash
# ❌ ทำงานบน main branch โดยตรง
git checkout main
git add .
git commit -m "fix"
git push origin main

# ❌ สร้าง branch จาก branch ที่ไม่เหมาะสม
git checkout feature/old-feature
git checkout -b feature/new-feature

# ❌ ไม่ pull ล่าสุดก่อนสร้าง branch
git checkout -b feature/new-stuff  # ไม่ pull develop ก่อน
```

### Commit Message ที่ไม่ดี
```bash
# ไม่มีข้อมูล ไม่ชัดเจน
"fix"
"update"
"changes"
"work in progress"
"test 123"
"asdfgh"
```

## Best Practices

### 1. Branch Naming Convention
```bash
# รูปแบบ: type/description-with-dashes
feature/add-payment-gateway
bugfix/fix-login-validation
hotfix/security-patch
release/v1.2.0
chore/update-dependencies
```

### 2. Branch Protection Rules
- ห้าม push ตรงไปยัง `main` และ `develop`
- ต้องผ่าน Pull Request และ Code Review
- ต้องผ่าน CI/CD tests
- ต้องมี approver อย่างน้อย 1-2 คน

### 3. Merge Strategy
```bash
# ใช้ Squash Merge สำหรับ feature branches
git merge --squash feature/user-auth

# ใช้ Merge Commit สำหรับ release branches
git merge --no-ff release/v1.2.0
```

### 4. Branch Lifecycle
1. **สร้าง** - จาก base branch ที่เหมาะสม
2. **พัฒนา** - commit เป็นระยะ ๆ
3. **Update** - rebase หรือ merge จาก base branch
4. **Review** - สร้าง Pull Request
5. **Merge** - หลังผ่าน review และ tests
6. **Cleanup** - ลบ branch ที่ไม่ใช้แล้ว

## เทคนิคขั้นสูง

### 1. Rebase vs Merge
```bash
# Rebase - สร้าง linear history
git checkout feature/user-auth
git rebase develop

# Merge - เก็บ history ของ branch
git checkout develop
git merge feature/user-auth
```

### 2. Cherry-pick
```bash
# นำ commit เฉพาะมาใช้ใน branch อื่น
git cherry-pick <commit-hash>
```

### 3. Branch Cleanup
```bash
# ลบ remote branches ที่ถูก merge แล้ว
git branch -r --merged | grep -v main | xargs git push origin --delete

# ลบ local branches
git branch --merged | grep -v main | xargs git branch -d
```

## สรุป

การเลือกใช้ Git Branching Strategy ที่เหมาะสมจะช่วยให้:
- ทีมทำงานร่วมกันได้อย่างมีประสิทธิภาพ
- ลดความขัดแย้งใน code (merge conflicts)
- ควบคุม code quality ได้ดีขึ้น
- ติดตาม history และ changes ได้ชัดเจน
- Deploy production ได้อย่างปลอดภัย

**หลักสำคัญ:** เลือกกลยุทธ์ที่เหมาะกับทีมและโปรเจกต์ จัดทำ convention ที่ชัดเจน และปฏิบัติให้สอดคล้องกันทั้งทีม
