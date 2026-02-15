# 🔧 FixIt – Building Fault Management System

A client-server application for reporting and managing building/neighborhood faults. Residents report issues with photos and GPS location, building managers assign technicians, and technicians update repair status — all in real time.

> 📚 **CS Final Project (5 Units)** – Israel Ministry of Education, Computer Science & Software Engineering

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Database Schema](#database-schema)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Database Setup](#database-setup)
  - [Server Setup](#server-setup)
  - [Android App](#android-app)
  - [Admin Desktop App](#admin-desktop-app)
- [API Endpoints](#api-endpoints)
- [Screenshots](#screenshots)
- [Extensions](#extensions)
- [License](#license)

---

## Overview

**FixIt** streamlines the process of reporting and resolving building maintenance issues. Instead of phone calls, WhatsApp messages, and lost requests — everything goes through one system with full visibility for all parties.

### The Problem
- Residents have no easy way to report faults with evidence
- Building managers lose track of open issues
- Technicians have no centralized task list
- No visibility into resolution times or quality

### The Solution
A multi-platform system with role-based access:

| Role | Platform | What they do |
|------|----------|-------------|
| **Resident** | Android | Report faults (photo + GPS), track status, rate repairs |
| **Technician** | Android | View assigned tasks, update progress, upload completion photos |
| **Manager** | Desktop (JavaFX) | View all faults, assign technicians, manage priorities |
| **Admin** | Desktop (JavaFX) | Manage buildings, users, categories, view system stats |

---

## Features

### Core
- 🏠 Multi-building support with apartment-level granularity
- 👥 4-level role-based access control (Admin, Manager, Technician, Resident)
- 📝 Full fault lifecycle: New → Assigned → In Progress → Done → Closed
- 💬 Comments thread on each fault
- ⭐ Satisfaction rating system for completed repairs
- 🔐 Password hashing (SHA-256 + salt)
- 🛡️ SQL injection protection (PreparedStatement throughout)

### Extensions (Section 9 – Advanced)
- 📸 **Camera** – Photo upload (before/during/after) from Android to server
- 🗺️ **Maps** – GPS fault location, map view with color-coded priority markers

### Technical
- REST API with JSON communication
- Async network calls (Kotlin Coroutines + Retrofit)
- Server-side image storage with DB path references
- Complex SQL queries (JOINs, aggregations, subqueries)
- OOP with inheritance (User → Resident/Technician/Manager)

---

## Architecture

```
┌─────────────────────┐     ┌─────────────────────┐
│   Android App       │     │   JavaFX Desktop    │
│   (Kotlin)          │     │   Admin App         │
│                     │     │                     │
│  👤 Resident        │     │  👨‍💼 Admin           │
│  🔧 Technician      │     │  🏢 Manager          │
└──────────┬──────────┘     └──────────┬──────────┘
           │         HTTP/REST         │
           │          (JSON)           │
           └────────────┬──────────────┘
                        │
           ┌────────────▼──────────────┐
           │   Java Spring Boot        │
           │   REST API Server         │
           │                           │
           │   /api/auth               │
           │   /api/faults             │
           │   /api/users              │
           │   /api/buildings          │
           │   /api/images             │
           │   /api/comments           │
           │   /api/stats              │
           └────────────┬──────────────┘
                        │ JDBC
           ┌────────────▼──────────────┐
           │      SQL Server           │
           │      (FixItDB)            │
           └───────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Android Client | Kotlin, Android Studio | API 24+ |
| Admin Desktop | JavaFX | 17+ |
| Server | Java, Spring Boot | 17, 3.x |
| Database | Microsoft SQL Server | 2019+ |
| HTTP Client (Android) | Retrofit 2 + OkHttp | 2.9.x |
| Image Loading | Glide / Coil | Latest |
| Maps | Google Maps SDK for Android | Latest |
| Camera | CameraX / Intent | Latest |

---

## Database Schema

**9 tables** including 2 linking tables:

```
Users ──────────┐
  │ FK: RoleId  │
  ▼              │
Roles            │
                 │
Buildings ◄──────┘ (FK: BuildingId)
  │
  ▼
Faults ◄──── Categories (FK: CategoryId)
  │
  ├──► FaultAssignments  ──► Users (Technician)
  │        (linking table)
  ├──► FaultImages
  │
  └──► FaultComments

TechnicianSkills (linking table: Users ◄──► Categories)
```

Full CREATE scripts: [`database/create_tables.sql`](database/create_tables.sql)
Seed data: [`database/seed_data.sql`](database/seed_data.sql)

---

## Project Structure

```
FixIt/
│
├── database/
│   ├── create_tables.sql          # Table creation scripts
│   ├── seed_data.sql              # Sample data for testing
│   └── queries.sql                # Complex queries (for project book)
│
├── server/                        # Java Spring Boot REST API
│   └── src/main/java/com/fixit/
│       ├── FixItApplication.java
│       ├── model/                 # POJOs (User, Fault, Building, etc.)
│       ├── dao/                   # Data Access Objects (DB queries)
│       ├── service/               # Business logic
│       ├── controller/            # REST endpoints
│       └── util/                  # Helpers (password hashing, etc.)
│
├── android/                       # Kotlin Android App
│   └── app/src/main/
│       ├── java/com/fixit/
│       │   ├── api/               # Retrofit service + client
│       │   ├── model/             # Data classes
│       │   ├── ui/
│       │   │   ├── auth/          # Login, Register
│       │   │   ├── resident/      # Report fault, My faults
│       │   │   ├── technician/    # Assigned faults
│       │   │   └── common/        # Fault detail, Map view
│       │   └── util/              # Session, Location helpers
│       └── res/                   # Layouts, strings (Hebrew), drawables
│
├── admin-desktop/                 # JavaFX Admin/Manager App
│   └── src/main/java/com/fixit/admin/
│       ├── AdminApp.java
│       ├── api/                   # HTTP client to REST API
│       ├── controller/            # JavaFX controllers
│       └── view/                  # FXML files
│
├── docs/                          # Project documentation
│   ├── ERD.png
│   ├── UML_class_diagram.png
│   ├── screen_flow_diagram.png
│   └── project_book.pdf
│
└── README.md
```

---

## Getting Started

### Prerequisites

- **Java JDK 17+**
- **Android Studio** (latest stable)
- **SQL Server 2019+** (Express edition is fine)
- **SSMS** (SQL Server Management Studio)
- **Google Maps API Key** ([Get one here](https://developers.google.com/maps/documentation/android-sdk/get-api-key))

### Database Setup

1. Open SSMS and connect to your SQL Server instance
2. Create a new database called `FixItDB`
3. Run the table creation script:
   ```sql
   -- In SSMS: File → Open → database/create_tables.sql → Execute
   ```
4. Run the seed data script:
   ```sql
   -- In SSMS: File → Open → database/seed_data.sql → Execute
   ```

### Server Setup

1. Navigate to the server directory:
   ```bash
   cd server
   ```
2. Update `src/main/resources/application.properties` with your SQL Server credentials:
   ```properties
   spring.datasource.url=jdbc:sqlserver://localhost:1433;databaseName=FixItDB;encrypt=false
   spring.datasource.username=sa
   spring.datasource.password=YOUR_PASSWORD
   ```
3. Build and run:
   ```bash
   ./mvnw spring-boot:run
   ```
4. Verify: open `http://localhost:8080/api/categories` — should return JSON

### Android App

1. Open the `android/` folder in Android Studio
2. Add your Google Maps API key in `local.properties`:
   ```
   MAPS_API_KEY=your_key_here
   ```
3. Update the server URL in `RetrofitClient.kt`:
   ```kotlin
   private const val BASE_URL = "http://10.0.2.2:8080/api/"  // emulator
   // or your local IP for physical device
   ```
4. Build and run on emulator or device (API 24+)

### Admin Desktop App

1. Navigate to the admin desktop directory:
   ```bash
   cd admin-desktop
   ```
2. Update the server URL in the configuration
3. Build and run:
   ```bash
   ./mvnw javafx:run
   ```

---

## API Endpoints

### Authentication
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/login` | Login with username + password |
| POST | `/api/auth/register` | Register new resident |

### Faults
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/faults?buildingId=&status=&priority=` | List faults (filtered) |
| GET | `/api/faults/{id}` | Fault details with images & comments |
| POST | `/api/faults` | Report new fault |
| PUT | `/api/faults/{id}` | Update fault (status, priority) |
| PUT | `/api/faults/{id}/assign` | Assign fault to technician |
| PUT | `/api/faults/{id}/rate` | Rate completed repair (1-5) |

### Images
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/faults/{id}/images` | Upload fault image (multipart) |
| GET | `/api/images/{imageId}` | Retrieve image file |

### Comments
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/faults/{id}/comments` | List comments for fault |
| POST | `/api/faults/{id}/comments` | Add comment |

### Buildings, Categories, Users
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/buildings` | List all buildings |
| POST | `/api/buildings` | Create building (admin) |
| GET | `/api/categories` | List fault categories |
| GET | `/api/users?role=&buildingId=` | List users (filtered) |
| PUT | `/api/users/{id}` | Update user (admin) |

### Statistics
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/stats/building/{id}` | Building fault statistics |
| GET | `/api/stats/technicians` | Technician performance stats |

---

## Screenshots

> 📷 _Screenshots will be added as development progresses._

| Screen | Description |
|--------|-------------|
| Login | <!-- ![Login](docs/screenshots/login.png) --> |
| Report Fault | <!-- ![Report](docs/screenshots/report_fault.png) --> |
| Fault Map | <!-- ![Map](docs/screenshots/fault_map.png) --> |
| Manager Dashboard | <!-- ![Dashboard](docs/screenshots/manager_dashboard.png) --> |
| Technician Tasks | <!-- ![Tasks](docs/screenshots/technician_tasks.png) --> |

---

## Extensions

This project implements the following advanced extensions as required by the curriculum:

### Section 9 (High Level)
| Extension | Implementation |
|-----------|---------------|
| **📸 Camera / File Transfer** | Residents capture fault photos via Android camera, images uploaded to server via multipart REST, technicians upload "after" repair photos |
| **🗺️ Maps** | Google Maps integration showing fault locations with color-coded priority markers, GPS-based location capture when reporting faults |

### Additional (Section 10)
| Extension | Implementation |
|-----------|---------------|
| **🛡️ SQL Injection Protection** | All database queries use `PreparedStatement` with parameterized queries |

---

## Test Credentials (Seed Data)

| Username | Password | Role |
|----------|----------|------|
| `admin` | `admin123` | Admin |
| `manager1` | `manager123` | Manager |
| `tech1` | `tech123` | Technician |
| `resident1` | `resident123` | Resident |

> ⚠️ These are for development/testing only. Change before any deployment.

---

## License

This project is developed as a CS final project for educational purposes.