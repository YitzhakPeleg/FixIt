# FixIt – Building Fault Management System
## Full Implementation Plan

---

## Project Overview

**FixIt** is a client-server application for reporting and managing building/neighborhood faults. Residents report faults (with photos and GPS location), building managers assign them to technicians, and technicians update repair status.

### Tech Stack
| Layer | Technology |
|-------|-----------|
| Android Client | Kotlin, Android Studio |
| Admin Desktop Client | Java Swing/JavaFX (or Web – HTML+JS) |
| Server (API) | Java (Spring Boot REST API) |
| Database | SQL Server |
| Communication | REST (JSON over HTTP) |
| Image Storage | Server filesystem + DB path references |

### Platforms (2 required)
1. **Android app** – Resident + Technician interface
2. **Desktop app (JavaFX)** – Admin/Manager interface

### Extensions (Section 9 – high level)
1. **Camera** – photo upload from Android to server
2. **Maps** – GPS location of faults, displayed on map

### Permission Levels (4 roles)
| Role | Access |
|------|--------|
| Admin | Manage buildings, users, categories, view reports/stats |
| Manager (ועד בית) | View all faults in their building, assign to technician, change priority/status |
| Technician (בעל מקצוע) | View assigned faults, update status, upload "after" photos |
| Resident (דייר) | Report faults, view own faults, rate completed repairs |

---

## Database Design – SQL Server

### Tables

```sql
-- 1. Roles
CREATE TABLE Roles (
    RoleId INT PRIMARY KEY IDENTITY(1,1),
    RoleName NVARCHAR(50) NOT NULL  -- Admin, Manager, Technician, Resident
);

-- 2. Buildings
CREATE TABLE Buildings (
    BuildingId INT PRIMARY KEY IDENTITY(1,1),
    BuildingName NVARCHAR(100) NOT NULL,
    Address NVARCHAR(200) NOT NULL,
    City NVARCHAR(100) NOT NULL,
    Latitude FLOAT NULL,
    Longitude FLOAT NULL,
    NumberOfApartments INT NOT NULL
);

-- 3. Users
CREATE TABLE Users (
    UserId INT PRIMARY KEY IDENTITY(1,1),
    Username NVARCHAR(50) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(256) NOT NULL,  -- SHA-256 hashed
    Salt NVARCHAR(64) NOT NULL,
    FullName NVARCHAR(100) NOT NULL,
    Phone NVARCHAR(20) NULL,
    Email NVARCHAR(100) NULL,
    RoleId INT NOT NULL FOREIGN KEY REFERENCES Roles(RoleId),
    BuildingId INT NULL FOREIGN KEY REFERENCES Buildings(BuildingId),
    ApartmentNumber NVARCHAR(10) NULL,
    IsActive BIT NOT NULL DEFAULT 1,
    CreatedAt DATETIME NOT NULL DEFAULT GETDATE()
);

-- 4. Categories
CREATE TABLE Categories (
    CategoryId INT PRIMARY KEY IDENTITY(1,1),
    CategoryName NVARCHAR(50) NOT NULL,  -- plumbing, electrical, elevator, cleaning, garden, parking...
    Icon NVARCHAR(50) NULL               -- icon name for UI
);

-- 5. Faults
CREATE TABLE Faults (
    FaultId INT PRIMARY KEY IDENTITY(1,1),
    ReportedByUserId INT NOT NULL FOREIGN KEY REFERENCES Users(UserId),
    BuildingId INT NOT NULL FOREIGN KEY REFERENCES Buildings(BuildingId),
    CategoryId INT NOT NULL FOREIGN KEY REFERENCES Categories(CategoryId),
    Title NVARCHAR(200) NOT NULL,
    Description NVARCHAR(1000) NULL,
    Priority NVARCHAR(20) NOT NULL DEFAULT 'Medium',  -- Low, Medium, High, Critical
    Status NVARCHAR(20) NOT NULL DEFAULT 'New',        -- New, Assigned, InProgress, Done, Closed
    Latitude FLOAT NULL,
    Longitude FLOAT NULL,
    LocationDescription NVARCHAR(200) NULL,  -- e.g. "3rd floor hallway"
    ReportedAt DATETIME NOT NULL DEFAULT GETDATE(),
    ResolvedAt DATETIME NULL,
    SatisfactionRating INT NULL  -- 1-5, NULL until resident rates
);

-- 6. FaultAssignments (linking table – Faults <-> Technicians)
CREATE TABLE FaultAssignments (
    AssignmentId INT PRIMARY KEY IDENTITY(1,1),
    FaultId INT NOT NULL FOREIGN KEY REFERENCES Faults(FaultId),
    TechnicianId INT NOT NULL FOREIGN KEY REFERENCES Users(UserId),
    AssignedByUserId INT NOT NULL FOREIGN KEY REFERENCES Users(UserId),
    AssignedAt DATETIME NOT NULL DEFAULT GETDATE(),
    Notes NVARCHAR(500) NULL
);

-- 7. FaultImages
CREATE TABLE FaultImages (
    ImageId INT PRIMARY KEY IDENTITY(1,1),
    FaultId INT NOT NULL FOREIGN KEY REFERENCES Faults(FaultId),
    UploadedByUserId INT NOT NULL FOREIGN KEY REFERENCES Users(UserId),
    ImagePath NVARCHAR(500) NOT NULL,       -- server filesystem path
    ImageType NVARCHAR(20) NOT NULL,        -- Before, After, During
    UploadedAt DATETIME NOT NULL DEFAULT GETDATE()
);

-- 8. FaultComments
CREATE TABLE FaultComments (
    CommentId INT PRIMARY KEY IDENTITY(1,1),
    FaultId INT NOT NULL FOREIGN KEY REFERENCES Faults(FaultId),
    UserId INT NOT NULL FOREIGN KEY REFERENCES Users(UserId),
    CommentText NVARCHAR(1000) NOT NULL,
    CreatedAt DATETIME NOT NULL DEFAULT GETDATE()
);

-- 9. TechnicianSkills (linking table – Technicians <-> Categories)
CREATE TABLE TechnicianSkills (
    TechnicianSkillId INT PRIMARY KEY IDENTITY(1,1),
    TechnicianId INT NOT NULL FOREIGN KEY REFERENCES Users(UserId),
    CategoryId INT NOT NULL FOREIGN KEY REFERENCES Categories(CategoryId),
    CONSTRAINT UQ_TechSkill UNIQUE (TechnicianId, CategoryId)
);
```

**Total: 9 tables including 2 linking tables (FaultAssignments, TechnicianSkills)**

### Key Queries (examples for the project book)

```sql
-- Open faults by building, category, priority
SELECT b.BuildingName, c.CategoryName, f.Priority, COUNT(*) AS OpenCount
FROM Faults f
JOIN Buildings b ON f.BuildingId = b.BuildingId
JOIN Categories c ON f.CategoryId = c.CategoryId
WHERE f.Status NOT IN ('Done','Closed')
GROUP BY b.BuildingName, c.CategoryName, f.Priority
ORDER BY OpenCount DESC;

-- Average resolution time per technician
SELECT u.FullName,
       AVG(DATEDIFF(HOUR, fa.AssignedAt, f.ResolvedAt)) AS AvgHours,
       COUNT(*) AS Completed
FROM FaultAssignments fa
JOIN Faults f ON fa.FaultId = f.FaultId
JOIN Users u ON fa.TechnicianId = u.UserId
WHERE f.Status = 'Done'
GROUP BY u.FullName;

-- Best available technician for a category (by current workload)
SELECT u.FullName, u.Phone,
       COUNT(CASE WHEN f2.Status IN ('Assigned','InProgress') THEN 1 END) AS CurrentLoad
FROM Users u
JOIN TechnicianSkills ts ON u.UserId = ts.TechnicianId
LEFT JOIN FaultAssignments fa ON u.UserId = fa.TechnicianId
LEFT JOIN Faults f2 ON fa.FaultId = f2.FaultId
WHERE ts.CategoryId = @categoryId AND u.IsActive = 1
GROUP BY u.FullName, u.Phone
ORDER BY CurrentLoad ASC;

-- Satisfaction rating per technician
SELECT u.FullName, AVG(CAST(f.SatisfactionRating AS FLOAT)) AS AvgRating, COUNT(*) AS Rated
FROM Faults f
JOIN FaultAssignments fa ON f.FaultId = fa.FaultId
JOIN Users u ON fa.TechnicianId = u.UserId
WHERE f.SatisfactionRating IS NOT NULL
GROUP BY u.FullName;
```

---

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐
│  Android App    │     │  JavaFX Admin   │
│  (Kotlin)       │     │  Desktop App    │
│  - Resident     │     │  - Admin        │
│  - Technician   │     │  - Manager      │
└────────┬────────┘     └────────┬────────┘
         │   HTTP/REST (JSON)    │
         └──────────┬────────────┘
                    │
         ┌──────────▼──────────┐
         │   Java Spring Boot  │
         │   REST API Server   │
         │                     │
         │  /api/auth          │
         │  /api/faults        │
         │  /api/users         │
         │  /api/buildings     │
         │  /api/images        │
         │  /api/comments      │
         │  /api/stats         │
         └──────────┬──────────┘
                    │ JDBC
         ┌──────────▼──────────┐
         │    SQL Server DB    │
         └─────────────────────┘
```

---

## Implementation Steps

Each step is a self-contained milestone. Complete and test each before moving to the next.

---

### STEP 1: Database Setup
**Goal:** SQL Server database with all tables, seed data, and tested queries.

**Tasks:**
- Install SQL Server (Express is fine) + SSMS
- Run the CREATE TABLE scripts above
- Insert seed data:
  - 4 roles
  - 3-4 buildings
  - 8-10 categories (plumbing, electrical, elevator, cleaning, garden, parking, general, security)
  - 10+ users across all roles
  - 15-20 sample faults in various statuses
- Test all complex queries in SSMS
- Document the ERD diagram (for project book)

**Deliverable:** Working database with seed data. All queries return correct results.

---

### STEP 2: Java Server – Project Setup + Models
**Goal:** Spring Boot project skeleton with all model classes (POJOs).

**Tasks:**
- Create Spring Boot project (Maven or Gradle)
  - Dependencies: `spring-boot-starter-web`, `mssql-jdbc`, `spring-boot-starter-data-jpa` (or plain JDBC)
- Create Java model classes matching each table:
  - `User.java`, `Role.java`, `Building.java`, `Category.java`
  - `Fault.java`, `FaultAssignment.java`, `FaultImage.java`, `FaultComment.java`
  - `TechnicianSkill.java`
- Configure `application.properties` for SQL Server connection:
  ```properties
  spring.datasource.url=jdbc:sqlserver://localhost:1433;databaseName=FixItDB
  spring.datasource.username=sa
  spring.datasource.password=YOUR_PASSWORD
  spring.datasource.driver-class-name=com.microsoft.sqlserver.cj.jdbc.Driver
  ```
- Verify server starts and connects to DB

**Deliverable:** Server starts, connects to SQL Server, model classes compile.

**Project structure:**
```
server/
├── src/main/java/com/fixit/
│   ├── FixItApplication.java
│   ├── model/
│   │   ├── User.java
│   │   ├── Role.java
│   │   ├── Building.java
│   │   ├── Category.java
│   │   ├── Fault.java
│   │   ├── FaultAssignment.java
│   │   ├── FaultImage.java
│   │   ├── FaultComment.java
│   │   └── TechnicianSkill.java
│   ├── dao/          (Data Access Objects – DB queries)
│   ├── service/      (Business logic)
│   ├── controller/   (REST endpoints)
│   └── util/         (Helpers – password hashing, etc.)
└── src/main/resources/
    └── application.properties
```

---

### STEP 3: Server – Authentication API
**Goal:** Login/Register endpoints with password hashing.

**Tasks:**
- Create `AuthController.java`:
  - `POST /api/auth/login` – accepts username+password, returns user object + role (or token)
  - `POST /api/auth/register` – creates new user (Resident role only for self-registration)
- Create `UserDAO.java`:
  - `getUserByUsername(String username)`
  - `createUser(User user)`
  - `getUsersByRole(int roleId)`
  - `getUsersByBuilding(int buildingId)`
- Create `PasswordUtil.java`:
  - `hashPassword(String password, String salt)` – SHA-256
  - `generateSalt()`
  - `verifyPassword(String password, String salt, String hash)`
- Use **PreparedStatement** everywhere (SQL injection protection)
- Test with Postman

**Deliverable:** Can register and login via Postman. Passwords stored hashed.

---

### STEP 4: Server – Fault CRUD API
**Goal:** Full fault management endpoints.

**Tasks:**
- Create `FaultController.java`:
  - `POST /api/faults` – create new fault
  - `GET /api/faults?buildingId=X&status=Y` – list faults with filters
  - `GET /api/faults/{id}` – fault details (with images, comments, assignment)
  - `PUT /api/faults/{id}` – update fault (status, priority)
  - `PUT /api/faults/{id}/assign` – assign to technician
  - `PUT /api/faults/{id}/rate` – resident rates completed fault
- Create `FaultDAO.java` with all queries (using JOINs for related data)
- Create `FaultService.java`:
  - Business logic: validate status transitions (New→Assigned→InProgress→Done→Closed)
  - Permission checks: only manager can assign, only technician can mark InProgress/Done
- Test all endpoints with Postman

**Deliverable:** Full fault lifecycle works via API.

---

### STEP 5: Server – Image Upload API
**Goal:** Upload and retrieve fault photos (Extension: Camera).

**Tasks:**
- Create `ImageController.java`:
  - `POST /api/faults/{id}/images` – multipart file upload
  - `GET /api/images/{imageId}` – serve image file
- Create upload directory on server (`/uploads/faults/{faultId}/`)
- Save image to filesystem, store path in `FaultImages` table
- Validate: file type (jpg/png only), max size (5MB)
- Return image URL in fault detail responses

**Deliverable:** Can upload photo via Postman, retrieve it via URL.

---

### STEP 6: Server – Comments, Stats, Buildings & Categories APIs
**Goal:** Remaining endpoints.

**Tasks:**
- `CommentController.java`:
  - `POST /api/faults/{id}/comments`
  - `GET /api/faults/{id}/comments`
- `BuildingController.java`:
  - `GET /api/buildings`
  - `POST /api/buildings` (admin only)
  - `GET /api/buildings/{id}/faults`
- `CategoryController.java`:
  - `GET /api/categories`
- `StatsController.java`:
  - `GET /api/stats/building/{id}` – fault counts by status, category
  - `GET /api/stats/technicians` – performance stats
- `UserController.java`:
  - `GET /api/users?role=Technician&buildingId=X`
  - `PUT /api/users/{id}` (admin – activate/deactivate)

**Deliverable:** All server APIs complete. Full Postman collection tested.

---

### STEP 7: Android App – Project Setup + Login
**Goal:** Kotlin Android project with login screen and API communication.

**Tasks:**
- Create Android Studio project (Kotlin)
- Add dependencies:
  - `Retrofit` + `Gson` – HTTP client
  - `OkHttp` – logging interceptor
  - `Glide` or `Coil` – image loading
  - Google Maps SDK
  - CameraX or implicit camera intent
- Create `ApiService.kt` interface (Retrofit):
  ```kotlin
  @POST("auth/login")
  suspend fun login(@Body request: LoginRequest): Response<User>
  ```
- Create `RetrofitClient.kt` singleton
- Build Login screen:
  - Username + Password fields
  - Login button → call API → store user in SharedPreferences
  - Route to correct home screen based on role
- Build Register screen (for residents)

**Deliverable:** Android app can login and identify user role.

---

### STEP 8: Android – Resident Screens
**Goal:** Resident can report faults and view their history.

**Tasks:**
- **Report Fault screen:**
  - Category dropdown (loaded from API)
  - Title + Description fields
  - Priority selector
  - Location description text field
  - GPS button → get current location (Latitude/Longitude)
  - Camera button → take photo (CameraX / intent)
  - Submit → POST to /api/faults + upload image
- **My Faults screen:**
  - RecyclerView showing resident's faults
  - Color-coded status badges
  - Click → Fault Detail screen
- **Fault Detail screen:**
  - Full info + images + comments
  - If status = Done → show "Rate" button (1-5 stars)
  - Add comment option
- **Navigation:** Bottom navigation bar (Report / My Faults / Profile)

**Deliverable:** Resident can report fault with photo and GPS, view their faults, rate repairs.

---

### STEP 9: Android – Technician Screens
**Goal:** Technician can view and manage assigned faults.

**Tasks:**
- **Assigned Faults screen:**
  - RecyclerView with faults assigned to this technician
  - Filter by status (Assigned / InProgress / Done)
- **Fault Detail (technician view):**
  - Same detail screen but with action buttons:
    - "Start Work" → status = InProgress
    - "Mark Complete" → status = Done
    - Upload "After" photo
    - Add comment/notes
- Reuse components from Step 8 where possible

**Deliverable:** Technician can manage their assigned faults end-to-end.

---

### STEP 10: Android – Map View (Extension: Maps)
**Goal:** Display faults on Google Maps.

**Tasks:**
- Add Google Maps API key
- Create Map screen (Fragment with `SupportMapFragment`)
- Load open faults for the user's building
- Display markers:
  - Color by priority (green=Low, yellow=Medium, orange=High, red=Critical)
  - Marker click → show fault title + status in info window
  - Info window click → navigate to Fault Detail
- When reporting a fault:
  - Show map for precise location picking (tap to place pin)
  - Or use "Current Location" button with GPS

**Deliverable:** Faults displayed on map. Location selection works when reporting.

---

### STEP 11: JavaFX Desktop Admin App
**Goal:** Desktop application for Admin and Manager roles.

**Tasks:**
- Create JavaFX project
- Same `RetrofitClient` / `HttpClient` pattern to call the REST API
- **Login screen** (same API)
- **Admin screens:**
  - Manage Buildings (CRUD table)
  - Manage Users (CRUD table with role assignment, activate/deactivate)
  - Manage Categories
  - System-wide statistics dashboard
- **Manager screens:**
  - View all faults in their building (table with filters)
  - Assign fault to technician (dropdown of available technicians by skill)
  - Change priority
  - View statistics for their building
- Use TableView for data grids
- Use charts (JavaFX Charts) for statistics

**Deliverable:** Admin and Manager can manage the system from desktop.

---

### STEP 12: Integration Testing + Polish
**Goal:** End-to-end testing of all flows.

**Tasks:**
- Test complete flows:
  1. Resident reports fault with photo + GPS → Manager sees it → assigns technician → technician marks done → resident rates
  2. Admin creates building + users → manager manages faults
  3. Map view shows correct markers
- Fix bugs and edge cases:
  - No internet handling
  - Empty states (no faults yet)
  - Input validation on all forms
  - Permission enforcement (API + UI)
- UI polish:
  - Consistent colors, icons, fonts
  - Loading indicators
  - Error messages (Hebrew)
- Performance: test with 50+ faults

**Deliverable:** Fully working system, all flows tested.

---

### STEP 13: Project Book (תיק פרויקט)
**Goal:** Complete documentation per the rubric.

**Sections and weights:**

| Section | Weight | Content |
|---------|--------|---------|
| **Cover page** | 2% | Per template |
| **Table of contents** | 2% | Linked to chapters |
| **Formatting** | 8% | Uniform font, headers, page numbers, margins |
| **Introduction (מבוא)** | 5% | Goals, system description, timeline – at least 1 page |
| **System requirements + UI overview** | 5% | Permissions, requirements, constraints, how to run |
| **User guide (מדריך למשתמש)** | 10% | Screen diagram + screenshots of EVERY screen with explanation of each control |
| **Architecture** | 15% | Class diagrams, folder structure, flow diagrams, code snippets with explanations |
| **Database** | 10% | ERD, table descriptions, sample data for each table |
| **UML + ERD diagrams** | 10% | Class relationships, table relationships, explanations |
| **Implementation (code docs)** | 15% | Document each class: purpose, variables, methods. Detail key methods with flowcharts |
| **Personal summary (רפלקציה)** | 5% | What learned, strengths/weaknesses, future improvements – NOT just "thank you" |
| **Appendix (נספחים)** | 10% | Full code printout as TEXT (not screenshots), including XML layouts |
| **Bonus** | up to 10% | Complex topic, significant code volume |

**Deliverable:** Complete project book as Word/PDF.

---

## File Structure Summary

```
FixIt/
├── database/
│   ├── create_tables.sql
│   ├── seed_data.sql
│   └── queries.sql
│
├── server/                          (Java Spring Boot)
│   └── src/main/java/com/fixit/
│       ├── FixItApplication.java
│       ├── model/
│       │   ├── User.java
│       │   ├── Role.java
│       │   ├── Building.java
│       │   ├── Category.java
│       │   ├── Fault.java
│       │   ├── FaultAssignment.java
│       │   ├── FaultImage.java
│       │   ├── FaultComment.java
│       │   └── TechnicianSkill.java
│       ├── dao/
│       │   ├── UserDAO.java
│       │   ├── FaultDAO.java
│       │   ├── BuildingDAO.java
│       │   ├── CategoryDAO.java
│       │   ├── ImageDAO.java
│       │   └── CommentDAO.java
│       ├── service/
│       │   ├── AuthService.java
│       │   ├── FaultService.java
│       │   └── ImageService.java
│       ├── controller/
│       │   ├── AuthController.java
│       │   ├── FaultController.java
│       │   ├── BuildingController.java
│       │   ├── CategoryController.java
│       │   ├── ImageController.java
│       │   ├── CommentController.java
│       │   ├── UserController.java
│       │   └── StatsController.java
│       └── util/
│           ├── PasswordUtil.java
│           └── DatabaseUtil.java
│
├── android/                         (Kotlin)
│   └── app/src/main/
│       ├── java/com/fixit/
│       │   ├── api/
│       │   │   ├── ApiService.kt
│       │   │   └── RetrofitClient.kt
│       │   ├── model/               (data classes mirroring server)
│       │   ├── ui/
│       │   │   ├── auth/            (LoginActivity, RegisterActivity)
│       │   │   ├── resident/        (ReportFaultActivity, MyFaultsActivity)
│       │   │   ├── technician/      (AssignedFaultsActivity)
│       │   │   ├── common/          (FaultDetailActivity, MapActivity)
│       │   │   └── adapters/        (RecyclerView adapters)
│       │   └── util/
│       │       ├── SessionManager.kt
│       │       └── LocationHelper.kt
│       └── res/
│           ├── layout/              (XML layouts)
│           ├── values/              (strings.xml – Hebrew)
│           └── drawable/
│
├── admin-desktop/                   (JavaFX)
│   └── src/main/java/com/fixit/admin/
│       ├── AdminApp.java
│       ├── api/                     (HTTP client to REST API)
│       ├── controller/              (JavaFX controllers)
│       └── view/                    (FXML files)
│
└── docs/
    ├── ERD.png
    ├── UML_class_diagram.png
    ├── screen_flow_diagram.png
    └── project_book.docx
```

---

## Suggested Build Order & Timeline

| Step | Description | Est. Time |
|------|-------------|-----------|
| 1 | Database setup + seed data | 1-2 days |
| 2 | Server project + models | 1 day |
| 3 | Auth API (login/register + hashing) | 2-3 days |
| 4 | Fault CRUD API | 3-4 days |
| 5 | Image upload API | 2 days |
| 6 | Remaining APIs (comments, stats, etc.) | 2-3 days |
| 7 | Android setup + login | 2-3 days |
| 8 | Android resident screens | 4-5 days |
| 9 | Android technician screens | 2-3 days |
| 10 | Map integration | 2-3 days |
| 11 | JavaFX admin desktop | 4-5 days |
| 12 | Integration testing + polish | 3-4 days |
| 13 | Project book | 5-7 days |
| | **Total** | **~5-7 weeks** |

---

## Notes

- **OOP requirement:** The model classes with inheritance (e.g., `User` base class → `Resident`, `Technician`, `Manager` subclasses) fulfill the OOP + inheritance rubric requirement.
- **Async requirement:** Retrofit + Kotlin coroutines on Android + async HTTP on JavaFX fulfill the async programming requirement.
- **SQL Injection:** Using PreparedStatement in all DAOs automatically covers this. Worth mentioning explicitly in the project book as an additional extension.
- **Hebrew UI:** All user-facing strings in Android and JavaFX should be in Hebrew (`strings.xml` / resource bundles).
- **The server runs locally** during the exam – make sure to have a portable setup (or document how to start it).
