# ATS Backend

Production-ready Applicant Tracking System backend built with FastAPI. Covers the full recruitment lifecycle — resume parsing via external ML API, candidate pipeline management, interview scheduling with automated email notifications, and role-based access control.

## Architecture

```
Client Request
     │
     ▼
┌──────────┐     ┌──────────┐      ┌────────────┐      ┌──────────┐
│  Router  │───> │ Service  │────> │ Repository │────> │  MSSQL   │
│ (FastAPI)│     │ (Logic)  │      │(SQLAlchemy)│      │    DB    │
└──────────┘     └──────────┘      └────────────┘      └──────────┘
                      │
                      ├──▶ External ML Parser API (httpx)
                      └──▶ SMTP Email Service
```

Routers handle HTTP, Services contain business logic, Repositories manage database operations. All layers are fully async.

## Features

**Resume Parsing** — Upload PDF/DOCX/DOC → text extraction (PyMuPDF, python-docx, LibreOffice) → ML API returns structured data → normalized, validated, and stored with raw text + original file. Email-based deduplication (skip or replace). Auto-calculated work experience. Every upload audit-logged.

**Job & Candidate Pipeline** — Job postings with expiry dates. Bulk candidate assignment with soft-delete. State machine workflow with transition history:
```
Applied → Screening → Interview → Offered → Hired
                                     ↘ Rejected (from any stage)
                                     ↘ Withdrawn (from any stage)
```

**Interview Scheduling** — Conflict detection for both candidate and interviewer time slots. Supports Video Call / In-Person / custom modes via configurable dropdowns. Duration resolved from master data, end time auto-calculated. Past-scheduling prevention (IST timezone-aware). Bulk scheduling with partial success handling. Automated HTML emails to candidate + interviewer.

**Interview Feedback** — Upsert feedback with file upload, rating, and result tracking. Public feedback page (no auth) for external interviewers — includes interview history, resume download, and dropdown access.

**Resume Management** — CRUD with multi-field search (name, email, phone, skills). Completeness validation. File download with proper content-type headers. Job-aware filtering and pagination.

**RBAC** — Users → Roles → Menu-level permissions (is_view / is_editable). Permission resolution endpoint for frontend menu rendering.

**Admin Dashboard** — Aggregated recruitment statistics.

**Supporting Systems** — Configurable master dropdowns by category. Login audit trail. Resume audit log for every parse attempt.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | FastAPI (async) |
| ORM | SQLAlchemy 2.0 (async) + aioodbc |
| Database | MSSQL (SQLite fallback for dev) |
| Auth | JWT via python-jose, HTTPBearer |
| HTTP Client | httpx (async) |
| Text Extraction | PyMuPDF, python-docx, LibreOffice |
| Email | SMTP via smtplib (async wrapper) |
| Validation | Pydantic v2 + pydantic-settings |
| Server | Uvicorn |
| Package Manager | uv |

## Project Structure

```
app/
├── api/
│   ├── router.py              # Mounts all sub-routers
│   └── v1/
│       ├── auth/              # Login / Logout
│       ├── dashboard/         # Admin stats
│       ├── dropdown/          # Master dropdowns
│       ├── interview/         # Scheduling, feedback, public feedback page
│       ├── job/               # Jobs, candidate assignment, status workflow
│       ├── user/              # Users, roles, permissions
│       ├── parser_router.py   # Resume upload & parsing
│       └── resume_router.py   # Resume CRUD & search
├── auth/                      # JWT security
├── core/                      # App factory, config, database, logger
├── crud/                      # Repository layer
├── models/                    # SQLAlchemy ORM models
├── schemas/                   # Pydantic request/response models
├── services/                  # Business logic
│   ├── parser_service.py      # ML API integration + retry
│   ├── parser_normalizer.py   # Normalize ML output
│   ├── resume_service.py      # Resume CRUD + dedup
│   ├── experience_service.py  # Work experience calculation
│   ├── interview/             # Scheduling, email, feedback
│   ├── job/                   # Job mgmt + state machine
│   └── user/                  # User, role, permission logic
└── utils/
    └── extract_text.py        # PDF/DOCX/DOC extraction
scripts/
└── setup_db.sql               # DB schema + seed data
```

## API Endpoints

All authenticated routes under `/api/ats/` require a Bearer JWT token.

| Module | Prefix | Auth | Endpoints |
|--------|--------|------|-----------|
| Auth | `/auth` | No | `POST /login`, `POST /logout` |
| Parser | `/parser` | Yes | `POST /upload` |
| Resumes | `/resumes` | Yes | `GET /`, `GET /search-candidates`, `GET /{id}`, `GET /download/{id}`, `POST /`, `PATCH /{id}`, `DELETE /{id}` |
| Jobs | `/jobs` | Yes | `POST /create-job`, `GET /get-job/{job_id}` |
| Job Candidates | `/job-candidates` | Yes | `POST /add-candidates`, `GET /get-job-candidates/{job_id}`, `GET /stage-history/{id}`, `PATCH /update-status/{id}` |
| Interviews | `/interviews` | Yes | `POST /schedule-interview`, `POST /schedule-interviews-bulk`, `GET /interview`, `GET /get-candidates-by-job/{job_id}`, `GET /candidates/{id}/interview-history`, `GET /{id}/interviews` |
| Feedback | `/interviews` | No | `POST /{id}/create-feedback`, `GET /{id}/get-feedback` |
| Feedback Page | `/interviews/feedback-page` | No | Interview history, resume download, dropdowns |
| Users | `/users` | Yes | `GET /get-user`, `GET /get-user-permissions`, `POST /create-user` |
| User Roles | `/user-roles` | Yes | `GET /user-roles`, `POST /edit-user-roles` |
| Role Permissions | `/user-role-permissions` | Yes | `GET /{role_name}`, `POST /upsert` |
| Dropdowns | `/dropdowns` | Yes | `GET /{category}` |
| Dashboard | `/dashboard` | Yes | `GET /dashboard` |

## Setup

**Prerequisites:** Python >= 3.12, MSSQL with ODBC Driver 18 (or SQLite for dev), LibreOffice (optional, for `.doc` files)

**Install:**
```bash
pip install uv && uv sync
```

**Configure** — create `.env` in project root:
```env
JWT_SECRET_KEY=<your-secret-key>
MSSQL_URL=mssql+aioodbc://user:pass@host/dbname?driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes
PARSE_API_ENDPOINT=http://localhost:2700/parse-resume
SMTP_HOST=smtp-mail.outlook.com
SMTP_PORT=587
SMTP_USER=<your-email>
SMTP_PASSWORD=<your-password>
DEFAULT_FROM_EMAIL=noreply@example.com
```

**Database:**
```bash
sqlcmd -S <server> -d <database> -i scripts/setup_db.sql
```

**Run:**
```bash
uv run main.py
```

Server: `http://localhost:8010` — API docs: `http://localhost:8010/api/docs`
