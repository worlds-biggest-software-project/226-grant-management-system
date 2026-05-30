# Grant Management System — Phased Development Plan

> Project: 226-grant-management-system · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | Python 3.12+ | LLM-heavy features (proposal drafting, narrative generation, risk scoring) are best served by the Python ML/AI ecosystem; strong library support for XBRL, federal API clients, and financial calculations |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 generation satisfies the standards requirement; async support for long-running LLM calls and external API polling; Pydantic v2 for request/response validation aligned with JSON Schema Draft 2020-12 |
| Database | PostgreSQL 16 | Required for JSONB with GIN indexes (hybrid data model), UUID generation, row-level security for multi-tenancy, and range types; production-grade for compliance-critical financial data |
| ORM / query builder | SQLAlchemy 2.0 + Alembic | Mature async support; Alembic for versioned migrations critical in compliance domain; type-safe query construction |
| Task queue | Celery 5 + Redis | Async workloads: LLM proposal generation, Grants.gov polling, compliance deadline monitoring, federal report generation, email notifications |
| Cache / broker | Redis 7 | Celery broker, session cache, rate-limit counters for external API calls |
| Frontend | Next.js 15 (React 19) + Tailwind CSS 4 + shadcn/ui | Dashboard-heavy UI with complex forms (application builder, review scoring, budget tracking); server components for SEO on public applicant portal; shadcn/ui for accessible, WCAG 2.2-compliant components |
| Authentication | OAuth 2.0 / OpenID Connect via python-jose + Authlib | RFC 6749 compliance; supports local auth + SSO (Azure AD, Okta) for enterprise deployments per standards.md |
| File storage | S3-compatible (MinIO for self-hosted, AWS S3 for cloud) | Document management for proposals, budgets, audit reports; presigned URLs for secure download |
| Containerisation | Docker + Docker Compose | Self-hosted deployment option differentiates against SaaS incumbents; compose orchestrates API, worker, PostgreSQL, Redis, MinIO |
| Testing | pytest + pytest-asyncio + httpx (backend); Vitest + Playwright (frontend) | pytest for unit/integration; httpx.AsyncClient for API testing; Playwright for WCAG accessibility and E2E |
| Code quality | Ruff (lint + format), mypy (type checking), pre-commit | Ruff replaces Black + isort + flake8; mypy for type safety in financial calculations |
| Package manager | uv (backend), pnpm (frontend) | uv for fast, reproducible Python dependency resolution; pnpm for efficient node_modules |
| LLM integration | OpenAI SDK (GPT-4o) + Anthropic SDK (Claude) via LiteLLM | LiteLLM provides provider-agnostic interface; supports self-hosted models for data-sensitive deployments |
| Key libraries | httpx (async HTTP for federal APIs), python-jsonschema (form validation), openpyxl (Excel export), weasyprint (PDF reports), passlib + bcrypt (password hashing) |

---

### Project Structure

```
grant-management-system/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── backend/
│   ├── __init__.py
│   ├── main.py                          # FastAPI application factory
│   ├── config.py                        # Settings via pydantic-settings
│   ├── database.py                      # SQLAlchemy engine, session factory
│   ├── models/                          # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── tenant.py
│   │   ├── user.py
│   │   ├── organisation.py
│   │   ├── programme.py
│   │   ├── application.py
│   │   ├── review.py
│   │   ├── award.py
│   │   ├── financial.py
│   │   ├── compliance.py
│   │   ├── document.py
│   │   ├── notification.py
│   │   └── audit.py
│   ├── schemas/                         # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── tenant.py
│   │   ├── user.py
│   │   ├── organisation.py
│   │   ├── programme.py
│   │   ├── application.py
│   │   ├── review.py
│   │   ├── award.py
│   │   ├── financial.py
│   │   ├── compliance.py
│   │   └── document.py
│   ├── api/                             # FastAPI route modules
│   │   ├── __init__.py
│   │   ├── deps.py                      # Dependency injection (auth, db session, tenant)
│   │   ├── auth.py
│   │   ├── tenants.py
│   │   ├── users.py
│   │   ├── organisations.py
│   │   ├── programmes.py
│   │   ├── applications.py
│   │   ├── reviews.py
│   │   ├── awards.py
│   │   ├── financial.py
│   │   ├── compliance.py
│   │   ├── documents.py
│   │   ├── reports.py
│   │   └── prospecting.py
│   ├── services/                        # Business logic layer
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── application_service.py
│   │   ├── review_service.py
│   │   ├── award_service.py
│   │   ├── budget_service.py
│   │   ├── compliance_service.py
│   │   ├── notification_service.py
│   │   ├── document_service.py
│   │   ├── report_service.py
│   │   └── audit_service.py
│   ├── integrations/                    # External API clients
│   │   ├── __init__.py
│   │   ├── grants_gov.py
│   │   ├── usaspending.py
│   │   ├── sam_gov.py
│   │   ├── candid.py
│   │   └── llm_client.py
│   ├── tasks/                           # Celery task modules
│   │   ├── __init__.py
│   │   ├── celery_app.py
│   │   ├── notifications.py
│   │   ├── compliance_checks.py
│   │   ├── prospecting.py
│   │   ├── report_generation.py
│   │   └── ai_tasks.py
│   └── utils/
│       ├── __init__.py
│       ├── pagination.py
│       ├── filtering.py
│       └── export.py
├── migrations/                          # Alembic migration scripts
│   ├── env.py
│   └── versions/
├── tests/
│   ├── conftest.py                      # Fixtures: test DB, client, factories
│   ├── factories.py                     # Factory Boy model factories
│   ├── fixtures/                        # Test data files
│   │   ├── sample_application.json
│   │   ├── sample_budget.json
│   │   ├── grants_gov_response.json
│   │   └── sam_gov_response.json
│   ├── unit/
│   │   ├── test_budget_service.py
│   │   ├── test_compliance_service.py
│   │   ├── test_review_service.py
│   │   └── test_report_service.py
│   ├── integration/
│   │   ├── test_applications_api.py
│   │   ├── test_awards_api.py
│   │   ├── test_reviews_api.py
│   │   └── test_financial_api.py
│   └── e2e/
│       └── test_grant_lifecycle.py
├── frontend/
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   └── register/page.tsx
│   │   │   ├── (dashboard)/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx
│   │   │   │   ├── programmes/
│   │   │   │   ├── applications/
│   │   │   │   ├── reviews/
│   │   │   │   ├── awards/
│   │   │   │   ├── compliance/
│   │   │   │   ├── reports/
│   │   │   │   └── settings/
│   │   │   └── (applicant-portal)/
│   │   │       ├── layout.tsx
│   │   │       ├── opportunities/
│   │   │       ├── apply/
│   │   │       └── my-applications/
│   │   ├── components/
│   │   │   ├── ui/                      # shadcn/ui components
│   │   │   ├── forms/
│   │   │   │   ├── FormBuilder.tsx
│   │   │   │   ├── DynamicForm.tsx
│   │   │   │   └── BudgetEditor.tsx
│   │   │   ├── reviews/
│   │   │   ├── awards/
│   │   │   └── layout/
│   │   ├── lib/
│   │   │   ├── api-client.ts
│   │   │   ├── auth.ts
│   │   │   └── utils.ts
│   │   └── types/
│   │       └── index.ts
│   └── tests/
│       ├── components/
│       └── e2e/
└── docs/
    └── api/                             # Auto-generated OpenAPI spec
```

---

## Phase 1: Foundation & Project Scaffold

### Purpose
Establish the project skeleton, development tooling, database connection, multi-tenant data model, authentication system, and CI pipeline. After this phase, a developer can run the application locally, authenticate, and interact with a working API that enforces tenant isolation.

### Tasks

#### 1.1 — Project Initialisation & Tooling

**What**: Create the project structure, configure dependency management, linting, type checking, and Docker Compose for local development.

**Design**:

`pyproject.toml` (key sections):
```toml
[project]
name = "grant-management-system"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sqlalchemy[asyncio]>=2.0.30",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",
    "pydantic>=2.9.0",
    "pydantic-settings>=2.6.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "httpx>=0.28.0",
    "redis>=5.2.0",
    "celery[redis]>=5.4.0",
    "python-multipart>=0.0.18",
    "boto3>=1.35.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "httpx>=0.28.0",
    "factory-boy>=3.3.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "pre-commit>=4.0.0",
]
```

`docker-compose.yml`:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: gms
      POSTGRES_USER: gms
      POSTGRES_PASSWORD: gms_dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

  api:
    build: .
    command: uvicorn backend.main:app --host 0.0.0.0 --port 8000 --reload
    environment:
      DATABASE_URL: postgresql+asyncpg://gms:gms_dev_password@db:5432/gms
      REDIS_URL: redis://redis:6379/0
      S3_ENDPOINT: http://minio:9000
      S3_ACCESS_KEY: minioadmin
      S3_SECRET_KEY: minioadmin
      S3_BUCKET: gms-documents
      SECRET_KEY: dev-secret-key-change-in-production
    ports: ["8000:8000"]
    depends_on: [db, redis, minio]
    volumes: [".:/app"]

  worker:
    build: .
    command: celery -A backend.tasks.celery_app worker --loglevel=info
    environment:
      DATABASE_URL: postgresql+asyncpg://gms:gms_dev_password@db:5432/gms
      REDIS_URL: redis://redis:6379/0
    depends_on: [db, redis]
    volumes: [".:/app"]

volumes:
  pgdata:
  miniodata:
```

`backend/config.py`:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://gms:gms_dev_password@localhost:5432/gms"
    redis_url: str = "redis://localhost:6379/0"
    secret_key: str = "change-me"
    access_token_expire_minutes: int = 60
    refresh_token_expire_days: int = 30
    s3_endpoint: str = "http://localhost:9000"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    s3_bucket: str = "gms-documents"
    cors_origins: list[str] = ["http://localhost:3000"]
    log_level: str = "INFO"

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}
```

**Testing**:
- Unit: `Settings` loads from environment variables; missing required fields raise `ValidationError`
- Unit: `Settings` loads from `.env` file with correct precedence (env var > .env > default)
- Integration: `docker-compose up` starts all services; API responds on port 8000; PostgreSQL accepts connections on 5432

#### 1.2 — Database Connection & Migration Infrastructure

**What**: Configure SQLAlchemy async engine, session management, and Alembic for migration tracking.

**Design**:

`backend/database.py`:
```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from backend.config import Settings

settings = Settings()
engine = create_async_engine(settings.database_url, echo=False, pool_size=10, max_overflow=20)
async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

`alembic.ini` configuration with `sqlalchemy.url` sourced from environment variable. `migrations/env.py` imports all models from `backend.models` and uses async engine.

**Testing**:
- Integration: `alembic upgrade head` runs without errors on a clean database
- Integration: `alembic downgrade -1` followed by `alembic upgrade head` is idempotent
- Unit: `get_db()` yields a session and commits on success
- Unit: `get_db()` rolls back on exception

#### 1.3 — Multi-Tenant & Core Identity Models

**What**: Create SQLAlchemy models for `tenants`, `organisations`, `jurisdictions`, `users`, `roles`, and `user_roles` with the initial Alembic migration.

**Design**:

Based on Data Model Suggestion 1 (Entity-Centric Normalized Relational) for core identity tables, with tenant-scoped row-level security:

```python
# backend/models/tenant.py
import uuid
from sqlalchemy import String, Boolean
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, DeclarativeBase
from datetime import datetime

class Base(DeclarativeBase):
    pass

class Tenant(Base):
    __tablename__ = "tenants"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(200), nullable=False)
    slug: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    tenant_type: Mapped[str] = mapped_column(String(30), nullable=False)  # 'grantmaker', 'grantseeker', 'government'
    settings: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()", onupdate=datetime.utcnow)
```

```python
# backend/models/organisation.py
class Organisation(Base):
    __tablename__ = "organisations"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False, index=True)
    name: Mapped[str] = mapped_column(String(500), nullable=False)
    legal_name: Mapped[str | None] = mapped_column(String(500))
    uei: Mapped[str | None] = mapped_column(String(12), index=True)  # GSA Unique Entity Identifier
    ein: Mapped[str | None] = mapped_column(String(10))
    organisation_type: Mapped[str] = mapped_column(String(50), nullable=False)
    profit_structure: Mapped[str | None] = mapped_column(String(30))
    jurisdiction_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("jurisdictions.id"))
    physical_address: Mapped[dict | None] = mapped_column(JSONB)
    mailing_address: Mapped[dict | None] = mapped_column(JSONB)
    website: Mapped[str | None] = mapped_column(String(500))
    phone: Mapped[str | None] = mapped_column(String(30))
    sam_registration_status: Mapped[str | None] = mapped_column(String(30))
    sam_registration_expiry: Mapped[date | None]
    fiscal_year_end_month: Mapped[int] = mapped_column(default=12)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/models/user.py
class User(Base):
    __tablename__ = "users"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False, index=True)
    email: Mapped[str] = mapped_column(String(320), nullable=False)
    display_name: Mapped[str] = mapped_column(String(200), nullable=False)
    password_hash: Mapped[str | None] = mapped_column(String(255))
    auth_provider: Mapped[str] = mapped_column(String(30), default="local")
    auth_provider_id: Mapped[str | None] = mapped_column(String(500))
    is_active: Mapped[bool] = mapped_column(default=True)
    last_login_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
    __table_args__ = (UniqueConstraint("tenant_id", "email"),)

class Role(Base):
    __tablename__ = "roles"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    permissions: Mapped[list] = mapped_column(JSONB, default=list)
    is_system_role: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    __table_args__ = (UniqueConstraint("tenant_id", "name"),)

class UserRole(Base):
    __tablename__ = "user_roles"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), nullable=False)
    role_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("roles.id"), nullable=False)
    organisation_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("organisations.id"))
    granted_at: Mapped[datetime] = mapped_column(server_default="now()")
    granted_by: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    __table_args__ = (UniqueConstraint("user_id", "role_id", "organisation_id"),)
```

Seed data for system roles: `admin`, `programme_officer`, `reviewer`, `applicant`, `finance_officer`.

**Testing**:
- Integration: Migration creates all tables with correct columns, indexes, and constraints
- Unit: `Tenant` model validates required fields (`name`, `slug`, `tenant_type`)
- Unit: `User` unique constraint on `(tenant_id, email)` prevents duplicates
- Integration: Creating a user with a non-existent `tenant_id` raises `IntegrityError`

#### 1.4 — Authentication & Authorisation

**What**: JWT-based authentication with login, registration, token refresh, and role-based permission checking.

**Design**:

```python
# backend/services/auth_service.py
from passlib.context import CryptContext
from jose import jwt
from datetime import datetime, timedelta

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class AuthService:
    def __init__(self, db: AsyncSession, settings: Settings):
        self.db = db
        self.settings = settings

    async def register(self, tenant_id: UUID, email: str, password: str, display_name: str) -> User:
        """Create user with hashed password and default 'applicant' role."""
        ...

    async def authenticate(self, tenant_id: UUID, email: str, password: str) -> TokenPair:
        """Verify credentials, return access + refresh tokens."""
        ...

    async def refresh_token(self, refresh_token: str) -> TokenPair:
        """Validate refresh token, issue new token pair."""
        ...

    def create_access_token(self, user: User, roles: list[str]) -> str:
        """JWT with claims: sub=user_id, tenant_id, roles, permissions, exp."""
        payload = {
            "sub": str(user.id),
            "tenant_id": str(user.tenant_id),
            "roles": roles,
            "exp": datetime.utcnow() + timedelta(minutes=self.settings.access_token_expire_minutes),
        }
        return jwt.encode(payload, self.settings.secret_key, algorithm="HS256")

    def verify_password(self, plain: str, hashed: str) -> bool:
        return pwd_context.verify(plain, hashed)

    def hash_password(self, password: str) -> str:
        return pwd_context.hash(password)
```

```python
# backend/schemas/user.py
class UserRegister(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)
    display_name: str = Field(min_length=1, max_length=200)

class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int  # seconds
```

```python
# backend/api/deps.py
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    """Decode JWT, load user, verify tenant scope."""
    ...

def require_permission(permission: str):
    """Dependency factory that checks user has a specific permission."""
    async def check(user: User = Depends(get_current_user), db: AsyncSession = Depends(get_db)):
        user_permissions = await get_user_permissions(db, user.id)
        if permission not in user_permissions:
            raise HTTPException(status_code=403, detail=f"Permission '{permission}' required")
        return user
    return check
```

API endpoints:
- `POST /api/v1/auth/register` — register new user (within a tenant)
- `POST /api/v1/auth/login` — email/password login, returns `TokenPair`
- `POST /api/v1/auth/refresh` — refresh access token
- `GET /api/v1/auth/me` — current user profile with roles

**Testing**:
- Unit: `hash_password` produces bcrypt hash; `verify_password` returns `True` for correct password, `False` for wrong
- Unit: `create_access_token` produces valid JWT with correct claims and expiry
- Unit: Expired token raises `401 Unauthorized`
- Integration: `POST /auth/register` creates user with hashed password and default role
- Integration: `POST /auth/login` with correct credentials returns token pair
- Integration: `POST /auth/login` with wrong password returns 401
- Integration: `GET /auth/me` with valid token returns user profile
- Integration: `GET /auth/me` without token returns 401
- Unit: `require_permission("applications.review")` returns 403 for user without that permission

#### 1.5 — Audit Logging Middleware

**What**: Automatic audit trail recording for all state-changing API calls, stored in the `audit_log` table.

**Design**:

```python
# backend/models/audit.py
class AuditLog(Base):
    __tablename__ = "audit_log"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(nullable=False, index=True)
    user_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    action: Mapped[str] = mapped_column(String(30), nullable=False)  # 'create', 'update', 'delete', 'status_change', 'login'
    entity_type: Mapped[str] = mapped_column(String(50), nullable=False)
    entity_id: Mapped[uuid.UUID] = mapped_column(nullable=False)
    changes: Mapped[dict | None] = mapped_column(JSONB)  # {"field": "status", "old": "draft", "new": "submitted"}
    ip_address: Mapped[str | None] = mapped_column(String(45))
    user_agent: Mapped[str | None] = mapped_column(String(500))
    occurred_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/services/audit_service.py
class AuditService:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def log(
        self,
        tenant_id: UUID,
        user_id: UUID | None,
        action: str,
        entity_type: str,
        entity_id: UUID,
        changes: dict | None = None,
        ip_address: str | None = None,
        user_agent: str | None = None,
    ) -> AuditLog:
        entry = AuditLog(
            tenant_id=tenant_id, user_id=user_id, action=action,
            entity_type=entity_type, entity_id=entity_id,
            changes=changes, ip_address=ip_address, user_agent=user_agent,
        )
        self.db.add(entry)
        return entry

    async def get_entity_history(
        self, tenant_id: UUID, entity_type: str, entity_id: UUID
    ) -> list[AuditLog]:
        """Return chronological audit trail for a specific entity."""
        ...
```

The audit service is injected into all service methods that modify state. Changes are computed by diffing the SQLAlchemy model's `__dict__` before and after modification.

**Testing**:
- Unit: `AuditService.log` creates an `AuditLog` entry with correct fields
- Integration: Creating an organisation via API produces an audit log entry with action='create'
- Integration: Updating an organisation produces an audit log entry with before/after changes in JSONB
- Unit: `get_entity_history` returns entries in chronological order
- Integration: Audit log entries include IP address and user agent from the request

#### 1.6 — FastAPI Application Shell & Health Check

**What**: Wire up the FastAPI application with CORS, exception handlers, OpenAPI configuration, and a health check endpoint.

**Design**:

```python
# backend/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

def create_app() -> FastAPI:
    app = FastAPI(
        title="Grant Management System",
        version="0.1.0",
        description="AI-native grant lifecycle management platform",
        docs_url="/api/docs",
        openapi_url="/api/v1/openapi.json",
    )
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    # Register routers
    app.include_router(auth_router, prefix="/api/v1/auth", tags=["auth"])
    app.include_router(tenants_router, prefix="/api/v1/tenants", tags=["tenants"])
    app.include_router(users_router, prefix="/api/v1/users", tags=["users"])
    app.include_router(organisations_router, prefix="/api/v1/organisations", tags=["organisations"])
    return app

app = create_app()

@app.get("/api/v1/health")
async def health_check(db: AsyncSession = Depends(get_db)):
    await db.execute(text("SELECT 1"))
    return {"status": "healthy", "version": "0.1.0"}
```

**Testing**:
- Integration: `GET /api/v1/health` returns 200 with `{"status": "healthy"}`
- Integration: `GET /api/docs` returns Swagger UI
- Integration: `GET /api/v1/openapi.json` returns valid OpenAPI 3.1 JSON
- Integration: CORS headers are present on responses when `Origin` header is sent

---

## Phase 2: Core Grant Lifecycle — Programmes, Opportunities & Applications

### Purpose
Implement the foundational grant lifecycle: grantmakers create programmes and funding opportunities; applicants submit applications with customisable form data and budgets; the system tracks application status through the lifecycle. After this phase, the core apply-and-track workflow is functional end-to-end through the API.

### Tasks

#### 2.1 — Grant Programmes CRUD

**What**: API endpoints for creating, reading, updating, and listing grant programmes within a tenant.

**Design**:

```python
# backend/models/programme.py
class GrantProgramme(Base):
    __tablename__ = "grant_programmes"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False, index=True)
    funder_organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisations.id"), nullable=False)
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    programme_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'federal', 'state', 'foundation', 'corporate'
    aln: Mapped[str | None] = mapped_column(String(10), index=True)  # Assistance Listing Number
    funding_source: Mapped[str | None] = mapped_column(String(50))
    total_budget: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    currency: Mapped[str] = mapped_column(String(3), default="USD")
    fiscal_year: Mapped[int | None]
    is_active: Mapped[bool] = mapped_column(default=True)
    config: Mapped[dict] = mapped_column(JSONB, default=dict)  # application_schema, budget_template, review_config, compliance_rules
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/schemas/programme.py
class ProgrammeCreate(BaseModel):
    funder_organisation_id: UUID
    title: str = Field(min_length=1, max_length=500)
    description: str | None = None
    programme_type: Literal["federal", "state", "foundation", "corporate"]
    aln: str | None = Field(None, pattern=r"^\d{2}\.\d{3}$")  # e.g., "47.076"
    total_budget: Decimal | None = Field(None, ge=0)
    currency: str = "USD"
    fiscal_year: int | None = None
    config: dict = Field(default_factory=dict)

class ProgrammeResponse(BaseModel):
    id: UUID
    tenant_id: UUID
    funder_organisation_id: UUID
    title: str
    programme_type: str
    aln: str | None
    total_budget: Decimal | None
    currency: str
    fiscal_year: int | None
    is_active: bool
    config: dict
    created_at: datetime
    updated_at: datetime
    model_config = ConfigDict(from_attributes=True)

class ProgrammeList(BaseModel):
    items: list[ProgrammeResponse]
    total: int
    page: int
    page_size: int
```

API endpoints:
- `POST /api/v1/programmes` — create programme (permission: `programmes.create`)
- `GET /api/v1/programmes` — list programmes with filtering and pagination
- `GET /api/v1/programmes/{id}` — get programme detail
- `PATCH /api/v1/programmes/{id}` — update programme (permission: `programmes.update`)

Filtering supports: `programme_type`, `is_active`, `fiscal_year`, `funder_organisation_id`. Pagination via `page` and `page_size` query parameters (default 20, max 100).

**Testing**:
- Integration: `POST /programmes` with valid data returns 201 and programme with generated UUID
- Integration: `POST /programmes` without `programmes.create` permission returns 403
- Integration: `GET /programmes` returns paginated list scoped to current tenant
- Integration: `GET /programmes?programme_type=federal` filters correctly
- Integration: `GET /programmes/{id}` for another tenant's programme returns 404
- Unit: `ProgrammeCreate` validates ALN format (`47.076` valid, `47076` invalid)
- Integration: Creating a programme produces an audit log entry

#### 2.2 — Funding Opportunities CRUD

**What**: API endpoints for creating and managing funding opportunities within a programme, including status lifecycle management.

**Design**:

```python
# backend/models/programme.py (continued)
class FundingOpportunity(Base):
    __tablename__ = "funding_opportunities"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False, index=True)
    grant_programme_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("grant_programmes.id"), nullable=False)
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    eligibility_criteria: Mapped[str | None] = mapped_column(Text)
    opportunity_number: Mapped[str | None] = mapped_column(String(100))
    grants_gov_id: Mapped[str | None] = mapped_column(String(50))
    posted_date: Mapped[date | None]
    close_date: Mapped[datetime | None]
    expected_awards_count: Mapped[int | None]
    award_floor: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    award_ceiling: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    total_funding_available: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    status: Mapped[str] = mapped_column(String(30), default="draft")  # 'draft', 'posted', 'closed', 'archived'
    config_overrides: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
```

Status transitions: `draft -> posted -> closed -> archived`. Transition validation:
- `draft -> posted`: requires `title`, `description`, `close_date` to be set
- `posted -> closed`: automatic when `close_date` passes, or manual
- `closed -> archived`: manual only

API endpoints:
- `POST /api/v1/programmes/{programme_id}/opportunities` — create opportunity
- `GET /api/v1/opportunities` — list opportunities (filterable by status, programme, date range)
- `GET /api/v1/opportunities/{id}` — get opportunity detail with programme config merged
- `PATCH /api/v1/opportunities/{id}` — update opportunity
- `POST /api/v1/opportunities/{id}/publish` — transition from draft to posted
- `POST /api/v1/opportunities/{id}/close` — transition to closed

**Testing**:
- Integration: `POST /programmes/{id}/opportunities` creates opportunity linked to programme
- Integration: `POST /opportunities/{id}/publish` with missing `close_date` returns 422
- Integration: `POST /opportunities/{id}/publish` on already-posted opportunity returns 409
- Integration: `GET /opportunities/{id}` returns merged config (programme config + opportunity overrides)
- Unit: Status transition validator rejects invalid transitions (e.g., `draft -> archived`)

#### 2.3 — Application Submission with Dynamic Forms

**What**: Applicants create and submit applications with programme-specific form data, budget data, and document attachments. Form data is validated against the programme's JSON Schema.

**Design**:

```python
# backend/models/application.py
class Application(Base):
    __tablename__ = "applications"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False, index=True)
    funding_opportunity_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("funding_opportunities.id"), nullable=False)
    applicant_organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisations.id"), nullable=False)
    submitted_by: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    application_number: Mapped[str] = mapped_column(String(50), nullable=False)  # auto-generated: GMS-YYYY-NNNN
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    abstract: Mapped[str | None] = mapped_column(Text)
    requested_amount: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    project_start_date: Mapped[date | None]
    project_end_date: Mapped[date | None]
    status: Mapped[str] = mapped_column(String(30), default="draft")
        # 'draft', 'submitted', 'under_review', 'approved', 'declined', 'withdrawn'
    submitted_at: Mapped[datetime | None]
    decision_date: Mapped[date | None]
    decision_notes: Mapped[str | None] = mapped_column(Text)
    form_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    budget_data: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
```

Application number generation: `GMS-{YEAR}-{sequential_4_digit}` scoped per tenant per year.

```python
# backend/services/application_service.py
class ApplicationService:
    async def create_draft(
        self, tenant_id: UUID, opportunity_id: UUID, org_id: UUID, user_id: UUID, title: str
    ) -> Application:
        """Create draft application; validate opportunity is 'posted' and not past close_date."""
        ...

    async def save_form_data(self, application_id: UUID, form_data: dict) -> Application:
        """Validate form_data against programme's application_schema JSON Schema, then save."""
        ...

    async def save_budget_data(self, application_id: UUID, budget_data: dict) -> Application:
        """Validate budget_data against programme's budget_template, then save."""
        ...

    async def submit(self, application_id: UUID, user_id: UUID) -> Application:
        """Transition to 'submitted': validate all required fields, form_data, budget_data are complete."""
        ...

    async def withdraw(self, application_id: UUID, user_id: UUID) -> Application:
        """Transition to 'withdrawn' if status is 'draft' or 'submitted'."""
        ...
```

JSON Schema validation uses `jsonschema.validate()` against the programme's `config.application_schema`. If no schema is defined, form_data is accepted as-is.

API endpoints:
- `POST /api/v1/opportunities/{opportunity_id}/applications` — create draft application
- `GET /api/v1/applications` — list applications (filterable by status, opportunity, organisation)
- `GET /api/v1/applications/{id}` — get application detail
- `PATCH /api/v1/applications/{id}` — update draft application (title, abstract, dates)
- `PUT /api/v1/applications/{id}/form-data` — save programme-specific form data
- `PUT /api/v1/applications/{id}/budget` — save budget data
- `POST /api/v1/applications/{id}/submit` — submit application
- `POST /api/v1/applications/{id}/withdraw` — withdraw application

**Testing**:
- Integration: Creating application for a closed opportunity returns 409
- Integration: `PUT /applications/{id}/form-data` with data violating JSON Schema returns 422 with field-level errors
- Integration: `POST /applications/{id}/submit` with missing required form fields returns 422
- Integration: `POST /applications/{id}/submit` transitions status to 'submitted' and sets `submitted_at`
- Unit: Application number generator produces sequential numbers within a tenant-year scope
- Unit: `withdraw` on an 'approved' application raises `InvalidStateTransition`
- Integration: Submitting an application produces audit log entry with status change

#### 2.4 — Document Management

**What**: Upload, list, download, and attach documents to applications, awards, and other entities using S3-compatible storage.

**Design**:

```python
# backend/models/document.py
class Document(Base):
    __tablename__ = "documents"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False, index=True)
    uploaded_by: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), nullable=False)
    file_name: Mapped[str] = mapped_column(String(500), nullable=False)
    file_type: Mapped[str | None] = mapped_column(String(100))
    file_size_bytes: Mapped[int | None]
    storage_key: Mapped[str] = mapped_column(String(1000), nullable=False)
    document_type: Mapped[str | None] = mapped_column(String(50))  # 'proposal', 'budget', 'report', 'audit', 'attachment'
    owner_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'application', 'award', 'compliance_item'
    owner_id: Mapped[uuid.UUID] = mapped_column(nullable=False)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/services/document_service.py
class DocumentService:
    def __init__(self, db: AsyncSession, s3_client):
        ...

    async def upload(
        self, tenant_id: UUID, user_id: UUID, file: UploadFile,
        owner_type: str, owner_id: UUID, document_type: str
    ) -> Document:
        """Upload file to S3 at key: {tenant_id}/{owner_type}/{owner_id}/{uuid}_{filename}."""
        ...

    async def get_download_url(self, document_id: UUID, tenant_id: UUID) -> str:
        """Generate presigned S3 URL valid for 15 minutes."""
        ...

    async def delete(self, document_id: UUID, tenant_id: UUID) -> None:
        """Delete from S3 and database; record in audit log."""
        ...

    async def list_for_entity(self, owner_type: str, owner_id: UUID) -> list[Document]:
        ...
```

API endpoints:
- `POST /api/v1/documents/upload` — multipart upload with `owner_type`, `owner_id`, `document_type` fields
- `GET /api/v1/documents?owner_type=application&owner_id={id}` — list documents for entity
- `GET /api/v1/documents/{id}/download` — returns presigned URL (302 redirect)
- `DELETE /api/v1/documents/{id}` — delete document

File size limit: 50 MB. Allowed types: PDF, DOCX, XLSX, CSV, PNG, JPG.

**Testing**:
- Integration: Upload a PDF; verify it appears in `list_for_entity`
- Integration: Download URL is a valid presigned S3 URL
- Integration: Uploading a 60 MB file returns 413
- Integration: Uploading an .exe file returns 415
- Integration: Deleting a document removes it from S3 and database
- Unit: Storage key format is `{tenant_id}/{owner_type}/{owner_id}/{uuid}_{filename}`

#### 2.5 — Notification System

**What**: Email and in-app notifications for application status changes, deadline reminders, and review assignments.

**Design**:

```python
# backend/models/notification.py
class Notification(Base):
    __tablename__ = "notifications"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(nullable=False, index=True)
    recipient_user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), nullable=False, index=True)
    notification_type: Mapped[str] = mapped_column(String(50), nullable=False)
    subject: Mapped[str] = mapped_column(String(500), nullable=False)
    body: Mapped[str | None] = mapped_column(Text)
    related_type: Mapped[str | None] = mapped_column(String(50))
    related_id: Mapped[uuid.UUID | None]
    is_read: Mapped[bool] = mapped_column(default=False)
    sent_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/services/notification_service.py
class NotificationService:
    async def create_notification(
        self, tenant_id: UUID, recipient_id: UUID,
        notification_type: str, subject: str, body: str,
        related_type: str | None = None, related_id: UUID | None = None,
    ) -> Notification:
        ...

    async def send_email(self, to_email: str, subject: str, body_html: str) -> None:
        """Send via configured SMTP or SES. Enqueued as Celery task."""
        ...

    async def get_unread(self, user_id: UUID, tenant_id: UUID) -> list[Notification]:
        ...

    async def mark_read(self, notification_id: UUID, user_id: UUID) -> None:
        ...
```

Notification triggers (called from service layer):
- Application submitted: notify programme officers
- Application status changed: notify applicant
- Review assigned: notify reviewer
- Deadline approaching (7 days, 1 day): notify applicant (via Celery beat)

API endpoints:
- `GET /api/v1/notifications` — list current user's notifications
- `GET /api/v1/notifications/unread-count` — count of unread
- `PATCH /api/v1/notifications/{id}/read` — mark as read
- `PATCH /api/v1/notifications/read-all` — mark all as read

**Testing**:
- Integration: Submitting an application creates a notification for programme officers
- Integration: `GET /notifications` returns only current user's notifications
- Integration: `PATCH /notifications/{id}/read` sets `is_read=True`
- Unit: Email notification is enqueued as Celery task, not sent synchronously
- Unit: Notification body contains application title and status

---

## Phase 3: Review & Award Workflow

### Purpose
Implement the review and scoring workflow: programme officers assign reviewers, reviewers score applications against rubrics, and decisions (approve/decline) are recorded. Awards are created from approved applications with budget line items. After this phase, the full apply-review-award pipeline is functional.

### Tasks

#### 3.1 — Review Panel & Reviewer Assignment

**What**: Create review panels for funding opportunities, assign reviewers to applications, and manage reviewer status (assigned, in_progress, completed, recused).

**Design**:

```python
# backend/models/review.py
class ReviewPanel(Base):
    __tablename__ = "review_panels"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False)
    funding_opportunity_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("funding_opportunities.id"), nullable=False)
    name: Mapped[str] = mapped_column(String(200), nullable=False)
    review_type: Mapped[str] = mapped_column(String(30), nullable=False)  # 'blind', 'open', 'panel'
    status: Mapped[str] = mapped_column(String(30), default="draft")  # 'draft', 'active', 'completed'
    created_at: Mapped[datetime] = mapped_column(server_default="now()")

class ReviewAssignment(Base):
    __tablename__ = "review_assignments"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    review_panel_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("review_panels.id"), nullable=False)
    application_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("applications.id"), nullable=False)
    reviewer_user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), nullable=False)
    status: Mapped[str] = mapped_column(String(30), default="assigned")  # 'assigned', 'in_progress', 'completed', 'recused'
    assigned_at: Mapped[datetime] = mapped_column(server_default="now()")
    completed_at: Mapped[datetime | None]
    __table_args__ = (UniqueConstraint("review_panel_id", "application_id", "reviewer_user_id"),)
```

For blind reviews, the API must strip applicant organisation name and identifying details from the application data returned to reviewers.

API endpoints:
- `POST /api/v1/opportunities/{id}/review-panels` — create review panel
- `POST /api/v1/review-panels/{id}/assignments` — bulk-assign reviewers to applications
- `GET /api/v1/review-panels/{id}/assignments` — list all assignments in panel
- `POST /api/v1/review-assignments/{id}/recuse` — reviewer recuses themselves
- `GET /api/v1/my-reviews` — reviewer sees their assigned applications

**Testing**:
- Integration: Creating a review panel for an opportunity with review_type='blind' succeeds
- Integration: Bulk assignment creates one assignment per (application, reviewer) pair
- Integration: Duplicate assignment returns 409
- Integration: `GET /my-reviews` for blind panel does not include organisation name
- Integration: Recusing from a completed review returns 409
- Unit: Reviewer cannot be assigned to their own organisation's application

#### 3.2 — Scoring Rubrics & Review Submission

**What**: Define scoring criteria per review panel, allow reviewers to enter scores and comments, compute weighted averages.

**Design**:

```python
# backend/models/review.py (continued)
class ScoringRubric(Base):
    __tablename__ = "scoring_rubrics"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    review_panel_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("review_panels.id"), nullable=False)
    criterion_name: Mapped[str] = mapped_column(String(300), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    max_score: Mapped[Decimal] = mapped_column(Numeric(5, 2), nullable=False)
    weight: Mapped[Decimal] = mapped_column(Numeric(5, 4), default=Decimal("1.0"))
    display_order: Mapped[int] = mapped_column(nullable=False)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")

class ReviewScore(Base):
    __tablename__ = "review_scores"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    review_assignment_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("review_assignments.id"), nullable=False)
    scoring_rubric_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("scoring_rubrics.id"), nullable=False)
    score: Mapped[Decimal | None] = mapped_column(Numeric(5, 2))
    comments: Mapped[str | None] = mapped_column(Text)
    scored_at: Mapped[datetime] = mapped_column(server_default="now()")
    __table_args__ = (UniqueConstraint("review_assignment_id", "scoring_rubric_id"),)
```

```python
# backend/services/review_service.py
class ReviewService:
    async def submit_scores(
        self, assignment_id: UUID, scores: list[ScoreInput], reviewer_id: UUID
    ) -> ReviewAssignment:
        """
        Validate: each score <= max_score, all rubric criteria covered.
        Save scores, compute weighted average, transition assignment to 'completed'.
        """
        ...

    async def get_application_scores_summary(
        self, application_id: UUID, panel_id: UUID
    ) -> ApplicationScoreSummary:
        """
        Return per-criterion averages across all completed reviews,
        plus overall weighted average.
        """
        ...

    def compute_weighted_average(self, scores: list[ReviewScore], rubrics: list[ScoringRubric]) -> Decimal:
        """Sum(score * weight) / Sum(weight) for each criterion."""
        total = sum(s.score * r.weight for s, r in zip(scores, rubrics))
        weight_sum = sum(r.weight for r in rubrics)
        return total / weight_sum if weight_sum > 0 else Decimal("0")
```

API endpoints:
- `POST /api/v1/review-panels/{id}/rubrics` — define scoring criteria (bulk create)
- `GET /api/v1/review-panels/{id}/rubrics` — list criteria
- `PUT /api/v1/review-assignments/{id}/scores` — submit all scores for an assignment
- `GET /api/v1/applications/{id}/review-summary` — aggregated scores (programme officer view)

**Testing**:
- Unit: `compute_weighted_average` with scores [8, 7, 9] and weights [0.3, 0.25, 0.2] returns correct average
- Unit: Score exceeding `max_score` raises `ValidationError`
- Integration: Submitting scores transitions assignment to 'completed' and sets `completed_at`
- Integration: Submitting scores for already-completed assignment returns 409
- Integration: `review-summary` averages scores across all completed reviews
- Integration: Missing criterion in score submission returns 422

#### 3.3 — Application Decision & Award Creation

**What**: Programme officers approve or decline applications. Approved applications generate awards with initial budget line items populated from the application's budget data.

**Design**:

```python
# backend/models/award.py
class Award(Base):
    __tablename__ = "awards"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False, index=True)
    application_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("applications.id"))
    grant_programme_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("grant_programmes.id"), nullable=False)
    recipient_organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisations.id"), nullable=False)
    award_number: Mapped[str] = mapped_column(String(100), nullable=False)  # auto-generated: AWD-YYYY-NNNN
    federal_award_id: Mapped[str | None] = mapped_column(String(50))  # FAIN for federal awards
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    award_amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    obligated_amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=Decimal("0"))
    disbursed_amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=Decimal("0"))
    currency: Mapped[str] = mapped_column(String(3), default="USD")
    award_date: Mapped[date] = mapped_column(nullable=False)
    start_date: Mapped[date] = mapped_column(nullable=False)
    end_date: Mapped[date] = mapped_column(nullable=False)
    status: Mapped[str] = mapped_column(String(30), default="active")  # 'active', 'suspended', 'closed', 'terminated'
    indirect_cost_rate: Mapped[Decimal | None] = mapped_column(Numeric(5, 4))
    indirect_cost_rate_type: Mapped[str | None] = mapped_column(String(30))  # 'negotiated', 'de_minimis', 'none'
    aln: Mapped[str | None] = mapped_column(String(10))
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")

class BudgetCategory(Base):
    __tablename__ = "budget_categories"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    code: Mapped[str] = mapped_column(String(20), unique=True, nullable=False)
    name: Mapped[str] = mapped_column(String(200), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    is_direct_cost: Mapped[bool] = mapped_column(nullable=False)
    parent_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("budget_categories.id"))
    display_order: Mapped[int] = mapped_column(nullable=False)

class BudgetLineItem(Base):
    __tablename__ = "budget_line_items"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    award_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("awards.id"), nullable=False, index=True)
    budget_category_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("budget_categories.id"), nullable=False)
    description: Mapped[str | None] = mapped_column(String(500))
    budgeted_amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    modified_amount: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    fiscal_year: Mapped[int | None]
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
```

Seed data for `budget_categories`: Personnel, Fringe Benefits, Travel, Equipment, Supplies, Contractual, Construction, Other Direct Costs, Indirect Costs (aligned with OMB 2 CFR Part 200 Subpart E).

```python
# backend/services/award_service.py
class AwardService:
    async def approve_application(
        self, application_id: UUID, approved_by: UUID,
        award_amount: Decimal, start_date: date, end_date: date,
        indirect_cost_rate: Decimal | None = None,
        decision_notes: str | None = None,
    ) -> Award:
        """
        1. Validate application status is 'submitted' or 'under_review'.
        2. Transition application to 'approved'.
        3. Create Award from application data.
        4. Create BudgetLineItems from application's budget_data.
        5. Create compliance obligations for federal awards.
        6. Notify applicant.
        7. Return Award.
        """
        ...

    async def decline_application(
        self, application_id: UUID, declined_by: UUID, decision_notes: str
    ) -> Application:
        """Transition to 'declined', notify applicant."""
        ...
```

API endpoints:
- `POST /api/v1/applications/{id}/approve` — approve and create award
- `POST /api/v1/applications/{id}/decline` — decline application
- `GET /api/v1/awards` — list awards (filterable by status, programme, recipient)
- `GET /api/v1/awards/{id}` — get award detail with budget line items
- `PATCH /api/v1/awards/{id}` — update award fields

**Testing**:
- Integration: Approving an application creates an award with matching amounts and dates
- Integration: Approving creates budget line items matching the application's budget_data
- Integration: Approving a 'draft' application returns 409 (must be submitted first)
- Integration: Declining an application transitions status and records decision_notes
- Integration: Award number follows format AWD-YYYY-NNNN
- Unit: For federal programmes (programme_type='federal'), approval auto-creates compliance obligations
- Integration: Applicant receives notification upon approval/decline

#### 3.4 — Award Amendments

**What**: Support budget modifications, no-cost extensions, and supplements to existing awards.

**Design**:

```python
# backend/models/award.py (continued)
class AwardAmendment(Base):
    __tablename__ = "award_amendments"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    award_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("awards.id"), nullable=False)
    amendment_number: Mapped[int] = mapped_column(nullable=False)
    amendment_type: Mapped[str] = mapped_column(String(50), nullable=False)
        # 'budget_modification', 'no_cost_extension', 'supplement', 'scope_change'
    description: Mapped[str] = mapped_column(Text, nullable=False)
    previous_amount: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    new_amount: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    previous_end_date: Mapped[date | None]
    new_end_date: Mapped[date | None]
    approved_by: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    approved_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
```

API endpoints:
- `POST /api/v1/awards/{id}/amendments` — create amendment
- `GET /api/v1/awards/{id}/amendments` — list amendments for award
- `POST /api/v1/amendments/{id}/approve` — approve amendment (applies changes to award)

When approved:
- `budget_modification` or `supplement`: updates `award.award_amount` and affected budget line items
- `no_cost_extension`: updates `award.end_date`
- `scope_change`: records the change; no automatic field update

**Testing**:
- Integration: Creating a no-cost extension amendment with new_end_date later than current end_date succeeds
- Integration: Approving a budget_modification updates the award's total amount
- Integration: Amendment number auto-increments per award (1, 2, 3...)
- Integration: Amendment creates audit log entry with before/after values
- Unit: Supplement with negative amount raises ValidationError

---

## Phase 4: Financial Management & Budget Tracking

### Purpose
Implement financial transaction recording, budget variance analysis, and expenditure tracking. After this phase, finance officers can record expenditures, track burn rates, and generate budget vs. actual reports per award.

### Tasks

#### 4.1 — Financial Transaction Recording

**What**: Record expenditures, drawdowns, reimbursements, and returns against award budget line items with allowability tracking per 2 CFR 200 Subpart E.

**Design**:

```python
# backend/models/financial.py
class FinancialTransaction(Base):
    __tablename__ = "financial_transactions"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    award_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("awards.id"), nullable=False, index=True)
    budget_line_item_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("budget_line_items.id"))
    transaction_type: Mapped[str] = mapped_column(String(30), nullable=False)
        # 'expenditure', 'drawdown', 'reimbursement', 'return'
    amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    transaction_date: Mapped[date] = mapped_column(nullable=False)
    description: Mapped[str | None] = mapped_column(String(500))
    reference_number: Mapped[str | None] = mapped_column(String(100))
    is_allowable: Mapped[bool | None]  # compliance flag per 2 CFR 200
    disallowance_reason: Mapped[str | None] = mapped_column(Text)
    posted_by: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/services/budget_service.py
class BudgetService:
    async def record_expenditure(
        self, award_id: UUID, budget_line_item_id: UUID,
        amount: Decimal, transaction_date: date, description: str,
        reference_number: str | None, posted_by: UUID,
    ) -> FinancialTransaction:
        """
        1. Validate award is 'active'.
        2. Validate budget_line_item belongs to award.
        3. Create transaction.
        4. Update award.disbursed_amount (sum of all expenditures).
        5. Check if expenditure exceeds budget line item (warn, don't block).
        6. Audit log.
        """
        ...

    async def get_budget_summary(self, award_id: UUID) -> BudgetSummary:
        """
        For each budget line item:
          budgeted, modified (or budgeted), expended (sum of transactions),
          remaining, variance_pct
        Plus totals: total_budgeted, total_expended, total_remaining, burn_rate
        """
        ...

    async def flag_disallowed(
        self, transaction_id: UUID, reason: str, flagged_by: UUID
    ) -> FinancialTransaction:
        """Mark a transaction as disallowed per 2 CFR 200."""
        ...
```

```python
# backend/schemas/financial.py
class BudgetSummary(BaseModel):
    award_id: UUID
    line_items: list[BudgetLineSummary]
    total_budgeted: Decimal
    total_expended: Decimal
    total_remaining: Decimal
    burn_rate_monthly: Decimal  # total_expended / months_elapsed
    months_remaining: int
    projected_total: Decimal  # burn_rate * total_months

class BudgetLineSummary(BaseModel):
    budget_line_item_id: UUID
    category_code: str
    category_name: str
    is_direct_cost: bool
    budgeted: Decimal
    modified: Decimal | None
    expended: Decimal
    disallowed: Decimal
    remaining: Decimal
    variance_pct: Decimal  # (expended - budgeted) / budgeted * 100
```

API endpoints:
- `POST /api/v1/awards/{id}/transactions` — record financial transaction
- `GET /api/v1/awards/{id}/transactions` — list transactions (filterable by type, date range, category)
- `GET /api/v1/awards/{id}/budget-summary` — budget vs. actual summary
- `POST /api/v1/transactions/{id}/flag-disallowed` — mark as disallowed
- `GET /api/v1/awards/{id}/transactions/export` — CSV export

**Testing**:
- Integration: Recording an expenditure updates `award.disbursed_amount`
- Integration: `budget-summary` computes correct variance percentages
- Unit: `burn_rate_monthly` calculation handles partial months and zero-expenditure periods
- Unit: Flagging a transaction as disallowed sets `is_allowable=False` with reason
- Integration: Recording transaction on a closed award returns 409
- Integration: CSV export includes all transaction fields with correct formatting

#### 4.2 — Budget Modification Workflow

**What**: Enable budget line item modifications with approval workflow, tracking the modification history.

**Design**:

Budget modifications are handled through award amendments (Phase 3.4). This task adds the budget-level detail:

```python
# backend/services/budget_service.py (continued)
class BudgetService:
    async def modify_budget_line(
        self, award_id: UUID, line_item_id: UUID,
        new_amount: Decimal, reason: str, modified_by: UUID,
    ) -> BudgetLineItem:
        """
        1. Store previous amount in `budgeted_amount`, set `modified_amount`.
        2. Check if total modified budget exceeds award_amount (warn).
        3. Create audit log entry.
        """
        ...

    async def check_budget_compliance(self, award_id: UUID) -> list[BudgetWarning]:
        """
        Check for:
        - Line items where expended > modified (overspent)
        - Total expenditure approaching award amount (>90%)
        - Indirect cost rate exceeds programme's cap
        - Individual category variance > 10% without amendment
        """
        ...
```

```python
class BudgetWarning(BaseModel):
    warning_type: str  # 'overspent', 'approaching_limit', 'rate_exceeded', 'variance_threshold'
    severity: str  # 'info', 'warning', 'critical'
    message: str
    details: dict
```

**Testing**:
- Unit: `check_budget_compliance` flags line item at 95% expenditure
- Unit: `check_budget_compliance` detects indirect cost rate exceeding 15% de minimis
- Integration: Modifying a budget line updates `modified_amount` and preserves `budgeted_amount`
- Integration: Modification creates audit log with before/after amounts

---

## Phase 5: Compliance, Subrecipients & Federal Reporting

### Purpose
Implement compliance obligation tracking, subrecipient monitoring per 2 CFR 200.332, and federal reporting (FFATA, SF-425). After this phase, the system supports the post-award compliance requirements that differentiate it from lighter-weight alternatives.

### Tasks

#### 5.1 — Compliance Obligation Management

**What**: Auto-generate and track compliance obligations per award based on programme rules (OMB 2 CFR Part 200 requirements, reporting deadlines, audit thresholds).

**Design**:

```python
# backend/models/compliance.py
class ComplianceRequirementType(Base):
    __tablename__ = "compliance_requirement_types"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    code: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)
    name: Mapped[str] = mapped_column(String(300), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    regulation_reference: Mapped[str | None] = mapped_column(String(200))
    frequency: Mapped[str | None] = mapped_column(String(30))  # 'one_time', 'quarterly', 'annual', 'as_needed'
    applies_to: Mapped[str | None] = mapped_column(String(30))  # 'all_federal', 'above_threshold', 'specific_programme'
    threshold_amount: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))

class ComplianceObligation(Base):
    __tablename__ = "compliance_obligations"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    award_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("awards.id"), nullable=False, index=True)
    requirement_type_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("compliance_requirement_types.id"), nullable=False)
    due_date: Mapped[date | None]
    status: Mapped[str] = mapped_column(String(30), default="pending")
        # 'pending', 'in_progress', 'submitted', 'accepted', 'overdue'
    completed_at: Mapped[datetime | None]
    notes: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
```

Seed data for compliance requirement types:
- `sf425_quarterly` — SF-425 Federal Financial Report (quarterly, all federal awards)
- `rppr` — Research Performance Progress Report (annual, research awards)
- `ffata_report` — FFATA/USASpending submission (one-time per new award + amendments > $25K)
- `single_audit` — Single Audit (annual, organisations spending >= $1M federal; per FY2026 threshold)
- `subrecipient_monitoring` — Subrecipient monitoring (per 2 CFR 200.332, as needed)
- `closeout` — Award closeout (one-time, within 120 days of end date)
- `equipment_inventory` — Equipment inventory (annual, for equipment purchased with federal funds)

```python
# backend/services/compliance_service.py
class ComplianceService:
    async def generate_obligations(self, award: Award) -> list[ComplianceObligation]:
        """
        Based on programme's compliance_rules config and award details:
        1. Determine which requirement types apply.
        2. Calculate due dates based on award dates and frequency.
        3. Create obligation records.
        For quarterly reports: generate 4 per year between start_date and end_date.
        For single audit: check if org's total federal expenditure >= $1M threshold.
        """
        ...

    async def check_overdue(self, tenant_id: UUID) -> list[ComplianceObligation]:
        """Find all obligations past due_date that are still 'pending' or 'in_progress'."""
        ...

    async def transition_status(
        self, obligation_id: UUID, new_status: str, user_id: UUID, notes: str | None
    ) -> ComplianceObligation:
        ...
```

Celery beat task: run `check_overdue()` daily, create notifications for obligations due in 7 days or overdue.

API endpoints:
- `GET /api/v1/awards/{id}/compliance` — list compliance obligations for award
- `PATCH /api/v1/compliance/{id}` — update obligation status/notes
- `GET /api/v1/compliance/dashboard` — cross-award compliance summary (pending, overdue, upcoming)

**Testing**:
- Unit: `generate_obligations` for a 2-year federal award creates correct number of quarterly SF-425 obligations
- Unit: Single audit obligation created only when org's federal expenditure >= $1M
- Integration: Celery task marks overdue obligations and creates notifications
- Integration: Compliance dashboard returns correct counts grouped by status
- Unit: Due date calculation for quarterly reports aligns with fiscal quarters

#### 5.2 — Subrecipient Tracking & Risk Assessment

**What**: Track subrecipients per 2 CFR 200.332, assess risk levels, and generate monitoring plans.

**Design**:

```python
# backend/models/compliance.py (continued)
class Subrecipient(Base):
    __tablename__ = "subrecipients"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    award_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("awards.id"), nullable=False, index=True)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisations.id"), nullable=False)
    subaward_number: Mapped[str] = mapped_column(String(100), nullable=False)
    subaward_amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    start_date: Mapped[date] = mapped_column(nullable=False)
    end_date: Mapped[date] = mapped_column(nullable=False)
    risk_level: Mapped[str] = mapped_column(String(20), default="standard")  # 'low', 'standard', 'high'
    risk_assessment_date: Mapped[date | None]
    risk_assessment_notes: Mapped[str | None] = mapped_column(Text)
    monitoring_frequency: Mapped[str | None] = mapped_column(String(30))  # 'quarterly', 'semi_annual', 'annual'
    status: Mapped[str] = mapped_column(String(30), default="active")
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
    updated_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/services/compliance_service.py (continued)
class ComplianceService:
    async def assess_subrecipient_risk(
        self, subrecipient_id: UUID, risk_factors: dict, assessed_by: UUID
    ) -> Subrecipient:
        """
        Risk factors considered (per 2 CFR 200.332):
        - Prior audit findings (material weakness, significant deficiency)
        - New vs returning subrecipient
        - Subaward amount relative to total award
        - Complexity of compliance requirements
        - SAM.gov registration status

        Scoring: sum weighted factors -> low (0-3), standard (4-6), high (7-10)
        Set monitoring_frequency based on risk_level.
        """
        ...
```

API endpoints:
- `POST /api/v1/awards/{id}/subrecipients` — add subrecipient
- `GET /api/v1/awards/{id}/subrecipients` — list subrecipients
- `POST /api/v1/subrecipients/{id}/risk-assessment` — perform risk assessment
- `GET /api/v1/subrecipients/{id}/monitoring-plan` — view monitoring requirements

**Testing**:
- Integration: Adding a subrecipient creates a record linked to both award and organisation
- Unit: Risk assessment with prior material weakness scores as 'high'
- Unit: Risk assessment for new subrecipient with large subaward scores as 'high'
- Unit: Monitoring frequency set to 'quarterly' for high-risk, 'annual' for low-risk
- Integration: Subrecipient with expired SAM registration generates warning notification

#### 5.3 — Federal Reporting (FFATA, SF-425)

**What**: Generate FFATA/USASpending submissions and SF-425 Federal Financial Reports from award and transaction data.

**Design**:

```python
# backend/models/compliance.py (continued)
class FederalReportSubmission(Base):
    __tablename__ = "federal_report_submissions"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False)
    report_type: Mapped[str] = mapped_column(String(50), nullable=False)  # 'ffata', 'sf425', 'sf270', 'rppr'
    reporting_period_start: Mapped[date] = mapped_column(nullable=False)
    reporting_period_end: Mapped[date] = mapped_column(nullable=False)
    status: Mapped[str] = mapped_column(String(30), default="draft")
    data: Mapped[dict] = mapped_column(JSONB, nullable=False)  # Report-specific GSDM-aligned data
    submitted_at: Mapped[datetime | None]
    submitted_by: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/services/report_service.py
class ReportService:
    async def generate_sf425(
        self, award_id: UUID, period_start: date, period_end: date
    ) -> FederalReportSubmission:
        """
        Generate SF-425 data from award + financial transactions:
        - Federal cash on hand
        - Total federal funds authorized
        - Federal share of expenditures (by category)
        - Unobligated balance
        - Indirect expense rate and amount
        All values sourced from financial_transactions within the period.
        """
        ...

    async def generate_ffata(self, award_id: UUID) -> FederalReportSubmission:
        """
        Generate FFATA submission data:
        - Award number (FAIN)
        - Recipient UEI, name, address
        - Award amount, action type
        - Project description (max 4000 chars)
        - Primary place of performance
        All values sourced from award + organisation data.
        """
        ...

    async def export_report(
        self, report_id: UUID, format: str = "json"
    ) -> bytes:
        """Export report in JSON, CSV, or PDF format."""
        ...
```

API endpoints:
- `POST /api/v1/awards/{id}/reports/sf425` — generate SF-425 for period
- `POST /api/v1/awards/{id}/reports/ffata` — generate FFATA submission
- `GET /api/v1/reports` — list reports (filterable by type, period, status)
- `GET /api/v1/reports/{id}` — view report detail
- `GET /api/v1/reports/{id}/export?format=json` — export report
- `POST /api/v1/reports/{id}/submit` — mark as submitted

**Testing**:
- Unit: SF-425 generation correctly sums expenditures by category within reporting period
- Unit: SF-425 unobligated balance = authorized - obligated
- Unit: FFATA data includes all required fields (UEI, FAIN, amount, description)
- Integration: Generating SF-425 creates a draft report linked to the award
- Integration: Exporting as JSON produces valid GSDM-aligned structure
- Unit: Project description truncated to 4000 characters for FFATA

#### 5.4 — Single Audit & SEFA Tracking

**What**: Track single audit requirements per 2 CFR 200 Subpart F and generate Schedule of Expenditures of Federal Awards (SEFA) data.

**Design**:

```python
# backend/models/compliance.py (continued)
class SingleAudit(Base):
    __tablename__ = "single_audits"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisations.id"), nullable=False)
    fiscal_year_end: Mapped[date] = mapped_column(nullable=False)
    total_federal_expenditure: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    audit_status: Mapped[str] = mapped_column(String(30), nullable=False)
        # 'required', 'in_progress', 'submitted', 'accepted'
    fac_submission_date: Mapped[date | None]  # Federal Audit Clearinghouse
    findings_count: Mapped[int] = mapped_column(default=0)
    material_weakness: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")

class SEFAEntry(Base):
    __tablename__ = "sefa_entries"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    single_audit_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("single_audits.id"), nullable=False)
    award_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("awards.id"), nullable=False)
    aln: Mapped[str] = mapped_column(String(10), nullable=False)
    programme_name: Mapped[str | None] = mapped_column(String(500))
    pass_through_entity: Mapped[str | None] = mapped_column(String(300))
    federal_expenditure: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    is_major_programme: Mapped[bool] = mapped_column(default=False)
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
```

```python
# backend/services/compliance_service.py (continued)
class ComplianceService:
    async def check_single_audit_required(
        self, organisation_id: UUID, fiscal_year_end: date
    ) -> bool:
        """
        Sum all federal expenditures across active awards for this org
        in the fiscal year. Return True if >= $1,000,000 (FY2026 threshold).
        """
        ...

    async def generate_sefa(
        self, organisation_id: UUID, fiscal_year_end: date
    ) -> SingleAudit:
        """
        Generate SEFA entries: group federal expenditures by ALN,
        determine major programme designation (Type A/B testing per 2 CFR 200.518).
        """
        ...
```

API endpoints:
- `GET /api/v1/organisations/{id}/single-audit-status` — check if audit is required
- `POST /api/v1/organisations/{id}/sefa` — generate SEFA for fiscal year
- `GET /api/v1/organisations/{id}/sefa/{fiscal_year}` — view SEFA schedule
- `PATCH /api/v1/single-audits/{id}` — update audit status/findings

**Testing**:
- Unit: Org with $1.2M federal expenditure triggers single audit requirement
- Unit: Org with $900K federal expenditure does not trigger requirement
- Unit: SEFA groups expenditures by ALN correctly
- Integration: SEFA generation creates entries for each federal award with correct ALN and amount
- Unit: Major programme determination follows Type A threshold (larger of $750K or 3% of total federal expenditure)

---

## Phase 6: Applicant Portal & Frontend Foundation

### Purpose
Build the Next.js frontend with authentication, applicant-facing portal (browse opportunities, submit applications, track status), and grantmaker dashboard shell. After this phase, both applicants and programme officers have a working web interface.

### Tasks

#### 6.1 — Frontend Project Setup & Auth

**What**: Initialise Next.js project with Tailwind CSS, shadcn/ui, API client, and JWT-based authentication flow.

**Design**:

```typescript
// frontend/src/lib/api-client.ts
class APIClient {
  private baseUrl: string;
  private accessToken: string | null;

  async login(email: string, password: string): Promise<TokenPair> { ... }
  async refreshToken(): Promise<TokenPair> { ... }
  async get<T>(path: string, params?: Record<string, string>): Promise<T> { ... }
  async post<T>(path: string, body: unknown): Promise<T> { ... }
  async patch<T>(path: string, body: unknown): Promise<T> { ... }
  async delete(path: string): Promise<void> { ... }

  // Automatic token refresh on 401
  private async handleUnauthorized(): Promise<void> { ... }
}
```

```typescript
// frontend/src/types/index.ts
interface User {
  id: string;
  email: string;
  display_name: string;
  roles: string[];
  tenant_id: string;
}

interface TokenPair {
  access_token: string;
  refresh_token: string;
  token_type: string;
  expires_in: number;
}

interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  page_size: number;
}
```

Pages:
- `/login` — email/password login form
- `/register` — registration form
- Layout with auth context provider, automatic redirect to `/login` for unauthenticated users

WCAG 2.2 Level AA compliance: all form inputs have labels, focus indicators are visible, colour contrast ratio >= 4.5:1, keyboard navigation works throughout.

**Testing**:
- E2E (Playwright): Login with valid credentials redirects to dashboard
- E2E: Login with wrong password shows error message
- E2E: Unauthenticated user accessing dashboard is redirected to login
- Component (Vitest): APIClient retries with refresh token on 401
- Accessibility: axe-core scan of login page reports zero violations

#### 6.2 — Applicant Portal: Browse & Apply

**What**: Public-facing pages for browsing open funding opportunities and submitting applications with dynamic forms.

**Design**:

Pages:
- `/opportunities` — list of posted funding opportunities with search and filters
- `/opportunities/{id}` — opportunity detail with eligibility criteria and "Apply" button
- `/apply/{opportunity_id}` — multi-step application form:
  - Step 1: Organisation selection/registration
  - Step 2: Project information (title, abstract, dates)
  - Step 3: Dynamic form fields (rendered from programme's `application_schema`)
  - Step 4: Budget entry (rendered from programme's `budget_template`)
  - Step 5: Document uploads
  - Step 6: Review & submit
- `/my-applications` — applicant's application list with status tracking

```typescript
// frontend/src/components/forms/DynamicForm.tsx
interface DynamicFormProps {
  schema: JSONSchema;         // From programme's application_schema
  data: Record<string, any>;  // Current form_data
  onChange: (data: Record<string, any>) => void;
  errors?: Record<string, string>;
}

// Renders form fields based on JSON Schema:
// - string -> text input or textarea (based on maxLength)
// - number -> numeric input with min/max
// - array -> repeatable section
// - enum -> select dropdown
// - boolean -> checkbox
```

```typescript
// frontend/src/components/forms/BudgetEditor.tsx
interface BudgetEditorProps {
  template: BudgetTemplate;    // From programme's budget_template
  data: BudgetData;
  onChange: (data: BudgetData) => void;
}

// Renders a budget table with:
// - Rows per budget category from template
// - Columns: Category, Description, Year 1, Year 2, ..., Total
// - Auto-calculated totals (direct, indirect, grand total)
// - Indirect cost calculation based on rate
```

Auto-save: form data is saved to the API every 30 seconds or on step navigation.

**Testing**:
- E2E: Browse opportunities -> select one -> fill application -> submit -> appears in my-applications
- E2E: Dynamic form renders correctly for a programme with custom fields
- E2E: Budget editor calculates totals correctly
- E2E: Application auto-saves on step navigation
- Accessibility: Multi-step form is navigable by keyboard; step indicators have ARIA labels
- Component: DynamicForm renders text, number, select, checkbox fields from JSON Schema

#### 6.3 — Grantmaker Dashboard Shell

**What**: Dashboard layout for programme officers with navigation, summary cards, and placeholder pages for each management area.

**Design**:

Dashboard layout:
- Sidebar navigation: Dashboard, Programmes, Applications, Reviews, Awards, Compliance, Reports, Settings
- Top bar: notifications bell with unread count, user menu
- Dashboard page with summary cards:
  - Applications pending review (count)
  - Awards active (count, total amount)
  - Compliance obligations overdue (count, red highlight)
  - Upcoming deadlines (next 7 days)

Pages (initial implementations with data tables):
- `/dashboard` — summary cards + recent activity
- `/dashboard/programmes` — list/manage programmes
- `/dashboard/applications` — list applications with status filters
- `/dashboard/awards` — list awards with budget summary
- `/dashboard/compliance` — compliance obligation dashboard

All data tables support: column sorting, text search, status filtering, pagination, and CSV export.

**Testing**:
- E2E: Programme officer sees dashboard with correct counts
- E2E: Applications table filters by status correctly
- E2E: Clicking an application navigates to detail view
- Accessibility: Sidebar navigation has correct ARIA roles; data tables have accessible headers
- Component: Summary card renders count and trend correctly

---

## Phase 7: External Integrations — Federal APIs

### Purpose
Connect to Grants.gov, SAM.gov, and USASpending.gov APIs for grant prospecting, entity validation, and federal reporting data. After this phase, the system can pull grant opportunities from federal databases and validate applicant registration status.

### Tasks

#### 7.1 — Grants.gov Integration

**What**: Client for the Grants.gov / Simpler.Grants.gov search API to import federal funding opportunities and synchronise opportunity data.

**Design**:

```python
# backend/integrations/grants_gov.py
class GrantsGovClient:
    BASE_URL = "https://api.simpler.grants.gov/v1"
    RATE_LIMIT = 60  # requests per minute

    def __init__(self, api_key: str | None = None, http_client: httpx.AsyncClient | None = None):
        self.api_key = api_key
        self.client = http_client or httpx.AsyncClient(timeout=30.0)

    async def search_opportunities(
        self,
        keywords: str | None = None,
        funding_instrument: str | None = None,  # 'grant', 'cooperative_agreement'
        agency: str | None = None,
        posted_after: date | None = None,
        close_after: date | None = None,
        page: int = 1,
        page_size: int = 25,
    ) -> GrantsGovSearchResult:
        """Search the Grants.gov opportunity database."""
        ...

    async def get_opportunity(self, opportunity_id: str) -> GrantsGovOpportunity:
        """Get full opportunity detail by Grants.gov ID."""
        ...

    async def sync_opportunities(
        self, tenant_id: UUID, programme_ids: list[UUID] | None = None
    ) -> SyncResult:
        """
        Periodic sync: fetch new/updated opportunities since last sync,
        match against saved searches, create opportunity_matches.
        """
        ...
```

```python
class GrantsGovOpportunity(BaseModel):
    opportunity_id: str
    opportunity_number: str
    title: str
    agency: str
    description: str
    funding_instrument: str
    category: str
    posted_date: date
    close_date: date | None
    award_floor: Decimal | None
    award_ceiling: Decimal | None
    expected_awards: int | None
    total_funding: Decimal | None
    eligibility: str | None
    aln: str | None
```

Celery periodic task: sync opportunities daily, respecting the 60 req/min rate limit.

**Testing**:
- Integration (mocked): `search_opportunities` with keywords returns parsed results
- Integration (mocked): `get_opportunity` returns full detail with correct field mapping
- Unit: Rate limiter prevents exceeding 60 requests/minute
- Integration (mocked): `sync_opportunities` creates `opportunity_matches` for matching saved searches
- Unit: GrantsGovSearchResult pagination handles multi-page results

#### 7.2 — SAM.gov Entity Validation

**What**: Validate applicant and subrecipient UEI registration status via the SAM.gov Entity API.

**Design**:

```python
# backend/integrations/sam_gov.py
class SAMGovClient:
    BASE_URL = "https://api.sam.gov/entity-information/v3/entities"

    def __init__(self, api_key: str, http_client: httpx.AsyncClient | None = None):
        self.api_key = api_key
        self.client = http_client or httpx.AsyncClient(timeout=30.0)

    async def validate_uei(self, uei: str) -> SAMEntityRecord | None:
        """
        Look up entity by UEI. Returns registration status,
        expiry date, exclusion status, business types.
        Returns None if UEI not found.
        """
        ...

    async def check_exclusions(self, uei: str) -> list[SAMExclusion]:
        """Check if entity has any active exclusions (debarment, suspension)."""
        ...

    async def bulk_validate(self, ueis: list[str]) -> dict[str, SAMEntityRecord | None]:
        """Validate multiple UEIs, respecting rate limits (10 req/day public, 1000/day registered)."""
        ...
```

Integration with organisation management: when a new organisation is created with a UEI, automatically validate against SAM.gov and store registration status and expiry. Celery task checks for expiring registrations (within 30 days) and sends notifications.

**Testing**:
- Integration (mocked): Valid UEI returns entity record with 'active' status
- Integration (mocked): Invalid UEI returns None
- Integration (mocked): Excluded entity returns exclusion records
- Unit: Bulk validation respects rate limits
- Integration: Organisation creation with UEI triggers SAM.gov validation

#### 7.3 — USASpending.gov Data Lookup

**What**: Query USASpending.gov API for award data used in FFATA reporting verification and federal expenditure lookups.

**Design**:

```python
# backend/integrations/usaspending.py
class USASpendingClient:
    BASE_URL = "https://api.usaspending.gov/api/v2"

    async def search_awards(
        self, recipient_uei: str | None = None,
        award_type: str = "grants",
        time_period: tuple[date, date] | None = None,
    ) -> list[USASpendingAward]:
        """Search federal awards by recipient or time period."""
        ...

    async def get_award_detail(self, award_id: str) -> USASpendingAwardDetail:
        """Get detailed award information including transaction history."""
        ...

    async def get_recipient_profile(self, uei: str) -> USASpendingRecipient:
        """Get recipient's federal award history."""
        ...
```

No authentication required (public API). Used for cross-referencing internal award data with federal records.

**Testing**:
- Integration (mocked): `search_awards` by UEI returns matching awards
- Integration (mocked): `get_recipient_profile` returns award history summary
- Unit: Response mapping handles missing optional fields gracefully

---

## Phase 8: Reporting & Analytics

### Purpose
Build reporting infrastructure for budget variance reports, programme analytics, and data export. After this phase, programme officers and finance officers can generate standardised reports and export data in multiple formats.

### Tasks

#### 8.1 — Standard Report Templates

**What**: Pre-built report templates for common grant management reports: budget vs. actual, award summary, application pipeline, compliance status.

**Design**:

```python
# backend/services/report_service.py (extended)
class ReportService:
    async def budget_vs_actual_report(
        self, award_id: UUID, as_of_date: date | None = None
    ) -> BudgetVsActualReport:
        """
        Per budget category: budgeted, modified, expended, encumbered, remaining.
        With variance percentages and burn rate projections.
        """
        ...

    async def programme_summary_report(
        self, programme_id: UUID, fiscal_year: int
    ) -> ProgrammeSummaryReport:
        """
        Aggregated stats: applications received, awarded, declined;
        total funding distributed; average award size; budget utilisation.
        """
        ...

    async def application_pipeline_report(
        self, opportunity_id: UUID
    ) -> ApplicationPipelineReport:
        """
        Applications by status (funnel): draft -> submitted -> under_review -> approved/declined.
        Average review scores, time-to-decision metrics.
        """
        ...

    async def compliance_status_report(
        self, tenant_id: UUID, as_of_date: date | None = None
    ) -> ComplianceStatusReport:
        """
        All compliance obligations grouped by status.
        Overdue items highlighted. Upcoming deadlines.
        """
        ...
```

Each report returns a structured Pydantic model and can be exported as JSON, CSV, or PDF (via WeasyPrint).

API endpoints:
- `GET /api/v1/reports/budget-vs-actual/{award_id}` — budget report
- `GET /api/v1/reports/programme-summary/{programme_id}?fiscal_year=2026` — programme report
- `GET /api/v1/reports/application-pipeline/{opportunity_id}` — pipeline report
- `GET /api/v1/reports/compliance-status` — compliance overview
- All endpoints accept `?format=json|csv|pdf` query parameter

**Testing**:
- Unit: Budget vs. actual correctly computes variance percentages
- Unit: Programme summary aggregates across all awards in the programme
- Unit: Pipeline report counts match application status distribution
- Integration: PDF export generates valid PDF with report data
- Integration: CSV export includes correct headers and data

#### 8.2 — Data Export & Scheduled Reports

**What**: Bulk data export for awards, transactions, and compliance data. Scheduled report generation via Celery beat.

**Design**:

```python
# backend/services/report_service.py (continued)
class ReportService:
    async def export_awards(
        self, tenant_id: UUID, filters: AwardFilters, format: str = "csv"
    ) -> bytes:
        """Export filtered awards data with budget summaries."""
        ...

    async def export_transactions(
        self, award_id: UUID, date_range: tuple[date, date], format: str = "csv"
    ) -> bytes:
        """Export financial transactions for an award within date range."""
        ...

    async def schedule_report(
        self, tenant_id: UUID, report_type: str, schedule: str,  # cron expression
        parameters: dict, recipients: list[UUID],
    ) -> ScheduledReport:
        """Create a recurring report that is generated and emailed on schedule."""
        ...
```

Celery beat schedules:
- Compliance deadline check: daily at 8:00 AM tenant-local time
- SAM.gov registration expiry check: weekly
- Overdue obligation escalation: daily

**Testing**:
- Integration: CSV export of awards includes all required columns
- Integration: Scheduled report is generated and emailed on schedule (mocked email)
- Unit: Date range filtering correctly includes boundary dates

---

## Phase 9: Grant Prospecting & Opportunity Matching

### Purpose
Build the grant prospecting feature: saved searches, ML-powered opportunity matching, and integration with Grants.gov and Candid APIs. After this phase, grantseekers can discover relevant funding opportunities matched to their organisation's profile and mission.

### Tasks

#### 9.1 — Saved Searches & Opportunity Database

**What**: Allow users to create saved searches with criteria (keywords, categories, amount ranges, deadlines) and store matched opportunities from external sources.

**Design**:

```python
# backend/models/prospecting.py (new file)
class SavedSearch(Base):
    __tablename__ = "saved_searches"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("tenants.id"), nullable=False)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id"), nullable=False)
    name: Mapped[str] = mapped_column(String(200), nullable=False)
    search_criteria: Mapped[dict] = mapped_column(JSONB, nullable=False)
    # search_criteria: {keywords: [], categories: [], amount_min, amount_max, deadline_after, agencies: [], funders: []}
    is_alert_active: Mapped[bool] = mapped_column(default=False)
    last_run_at: Mapped[datetime | None]
    created_at: Mapped[datetime] = mapped_column(server_default="now()")

class OpportunityMatch(Base):
    __tablename__ = "opportunity_matches"
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    saved_search_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("saved_searches.id"))
    organisation_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organisations.id"), nullable=False)
    external_opportunity_id: Mapped[str | None] = mapped_column(String(100))
    source: Mapped[str] = mapped_column(String(30), nullable=False)  # 'grants_gov', 'candid', 'manual'
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    funder_name: Mapped[str | None] = mapped_column(String(300))
    amount_low: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    amount_high: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    deadline: Mapped[datetime | None]
    relevance_score: Mapped[Decimal | None] = mapped_column(Numeric(5, 4))
    status: Mapped[str] = mapped_column(String(30), default="new")  # 'new', 'interested', 'applied', 'dismissed'
    created_at: Mapped[datetime] = mapped_column(server_default="now()")
```

API endpoints:
- `POST /api/v1/saved-searches` — create saved search
- `GET /api/v1/saved-searches` — list user's saved searches
- `POST /api/v1/saved-searches/{id}/run` — execute search now
- `GET /api/v1/opportunity-matches` — list matches (filterable by search, status, source)
- `PATCH /api/v1/opportunity-matches/{id}` — update match status (interested, dismissed)

**Testing**:
- Integration: Creating a saved search stores criteria correctly
- Integration: Running a saved search against Grants.gov (mocked) creates opportunity matches
- Integration: Dismissing a match updates status; dismissed matches are hidden from default listing
- Unit: Search criteria with date range correctly filters by deadline

#### 9.2 — ML-Powered Relevance Scoring

**What**: Score opportunity matches based on semantic similarity between the organisation's mission/past awards and the opportunity description, using LLM embeddings.

**Design**:

```python
# backend/services/prospecting_service.py
class ProspectingService:
    def __init__(self, llm_client, db: AsyncSession):
        self.llm_client = llm_client
        self.db = db

    async def compute_relevance_score(
        self, organisation_id: UUID, opportunity_text: str
    ) -> float:
        """
        1. Build organisation profile: mission statement, past award titles,
           programme areas, geographic focus.
        2. Generate embedding for org profile.
        3. Generate embedding for opportunity text.
        4. Compute cosine similarity.
        5. Return score in [0.0, 1.0].

        Embeddings are cached per organisation (refreshed when profile changes).
        """
        ...

    async def rank_opportunities(
        self, organisation_id: UUID, opportunities: list[OpportunityMatch]
    ) -> list[OpportunityMatch]:
        """Score and sort opportunities by relevance."""
        ...
```

LLM provider: use embedding model (e.g., `text-embedding-3-small` via OpenAI or equivalent). Embeddings are stored in a `organisation_embeddings` cache table with a staleness check.

**Testing**:
- Unit: Cosine similarity between identical texts returns ~1.0
- Unit: Cosine similarity between unrelated texts returns < 0.3
- Integration (mocked LLM): Opportunities matching org's mission score higher than unrelated ones
- Unit: Cached embeddings are reused when organisation profile hasn't changed

#### 9.3 — Candid Foundation Directory Integration

**What**: Client for the Candid Grants API to search foundation and corporate grant opportunities.

**Design**:

```python
# backend/integrations/candid.py
class CandidClient:
    BASE_URL = "https://api.candid.org/grants/v1"

    def __init__(self, api_key: str, http_client: httpx.AsyncClient | None = None):
        ...

    async def search_funders(
        self, keywords: str | None = None,
        geographic_focus: str | None = None,
        subject_area: str | None = None,
    ) -> list[CandidFunder]:
        """Search for foundations and corporate funders."""
        ...

    async def get_funder_grants(self, funder_ein: str) -> list[CandidGrant]:
        """Get recent grant transactions for a specific funder."""
        ...

    async def search_transactions(
        self, recipient_ein: str | None = None,
        subject: str | None = None,
        year: int | None = None,
    ) -> list[CandidGrant]:
        """Search grant transaction history."""
        ...
```

**Testing**:
- Integration (mocked): `search_funders` returns parsed funder records
- Integration (mocked): `get_funder_grants` returns grant transaction list
- Unit: API key is included in request headers

---

## Phase 10: AI-Native Features

### Purpose
Implement the AI-native differentiators: LLM-assisted proposal drafting, automated impact narrative generation, compliance monitoring agents, and subrecipient risk scoring. These features are the primary competitive advantage.

### Tasks

#### 10.1 — AI-Assisted Proposal Drafting

**What**: Generate first-draft proposal narratives from project descriptions, budget data, and programme requirements using LLM.

**Design**:

```python
# backend/services/ai_service.py
class AIService:
    def __init__(self, llm_client: LiteLLMClient):
        self.llm = llm_client

    async def draft_proposal_narrative(
        self, application_id: UUID, sections: list[str] | None = None
    ) -> ProposalDraft:
        """
        System prompt: "You are a grant writing expert. Generate a proposal
        narrative for a {programme_type} grant application."

        Context injected:
        - Programme description and eligibility criteria
        - Application form_data (project details, goals, population served)
        - Budget summary
        - Organisation mission and past awards

        Output: structured sections (Abstract, Statement of Need, Goals & Objectives,
        Methodology, Evaluation Plan, Sustainability, Budget Justification)

        Each section is generated separately to allow selective regeneration.
        """
        ...

    async def regenerate_section(
        self, application_id: UUID, section: str, instructions: str
    ) -> str:
        """Regenerate a specific section with user-provided instructions."""
        ...
```

```python
class ProposalDraft(BaseModel):
    application_id: UUID
    sections: dict[str, str]  # {"abstract": "...", "statement_of_need": "...", ...}
    model_used: str
    generated_at: datetime
    token_usage: dict  # {"prompt_tokens": N, "completion_tokens": M}
```

API endpoints:
- `POST /api/v1/applications/{id}/ai/draft-proposal` — generate full draft
- `POST /api/v1/applications/{id}/ai/regenerate-section` — regenerate one section
- `POST /api/v1/applications/{id}/ai/improve` — improve existing text with instructions

Async: proposal generation is a Celery task; API returns task ID for polling.

**Testing**:
- Integration (mocked LLM): Draft proposal returns structured sections
- Unit: System prompt includes programme type, eligibility criteria, and org mission
- Unit: Budget justification section references actual budget amounts
- Integration: Task status endpoint reports progress for long-running generation
- Unit: Section regeneration preserves other sections

#### 10.2 — Automated Impact Narrative Generation

**What**: Transform structured outcome data (milestones, participant counts, metrics) into grantmaker-ready narrative reports.

**Design**:

```python
# backend/services/ai_service.py (continued)
class AIService:
    async def generate_impact_narrative(
        self, award_id: UUID, reporting_period: tuple[date, date],
        outcome_data: dict,
    ) -> str:
        """
        Input: structured outcome data:
        {
            "participants_served": 1250,
            "milestones_completed": ["Curriculum developed", "Pilot program launched"],
            "metrics": [
                {"name": "Job placements", "target": 100, "actual": 87},
                {"name": "Training hours delivered", "target": 5000, "actual": 5400}
            ],
            "challenges": "Supply chain delays impacted equipment procurement...",
            "next_steps": "Expand to two additional counties..."
        }

        Output: Polished narrative report suitable for grantmaker submission.
        Tone: professional, data-driven, honest about challenges.
        Includes: executive summary, progress against objectives,
        key outcomes with contextual framing, challenges and mitigation,
        plans for next period.
        """
        ...
```

API endpoint:
- `POST /api/v1/awards/{id}/ai/impact-narrative` — generate narrative from outcome data

**Testing**:
- Integration (mocked LLM): Narrative includes all input metrics with contextual framing
- Unit: Narrative mentions actual vs. target performance
- Unit: Challenges section is included when challenges data is provided

#### 10.3 — AI-Powered Compliance Monitoring

**What**: Automated expenditure review that flags potentially disallowed costs per 2 CFR 200 Subpart E.

**Design**:

```python
# backend/services/ai_service.py (continued)
class AIService:
    async def review_expenditure_compliance(
        self, award_id: UUID, transaction_ids: list[UUID] | None = None,
    ) -> list[ComplianceFinding]:
        """
        For each expenditure transaction:
        1. Check description against 2 CFR 200 Subpart E cost principles.
        2. Compare category to allowable cost types for the programme.
        3. Flag potential issues:
           - Entertainment/alcohol expenses (generally disallowed)
           - Equipment purchases without prior approval
           - Travel costs exceeding federal per diem rates
           - Indirect costs exceeding negotiated rate
        4. Return findings with severity, regulation reference, and recommendation.
        """
        ...

class ComplianceFinding(BaseModel):
    transaction_id: UUID
    severity: str  # 'info', 'warning', 'critical'
    finding_type: str  # 'potentially_disallowed', 'missing_approval', 'rate_exceeded', 'documentation_needed'
    description: str
    regulation_reference: str  # e.g., '2 CFR 200.423 (Alcoholic beverages)'
    recommendation: str
```

Celery task: run compliance review weekly on new transactions.

**Testing**:
- Unit: Transaction with "alcohol" or "entertainment" in description flagged as potentially disallowed with reference to 2 CFR 200.423
- Unit: Equipment purchase > $5,000 without approval flag returns 'missing_approval'
- Unit: Indirect costs exceeding programme's rate cap returns 'rate_exceeded'
- Integration (mocked LLM): Batch review processes multiple transactions and returns findings

#### 10.4 — ML-Based Subrecipient Risk Scoring

**What**: Automated risk scoring for subrecipients based on financial health indicators, audit history, and reporting reliability.

**Design**:

```python
# backend/services/ai_service.py (continued)
class AIService:
    async def score_subrecipient_risk(
        self, subrecipient_id: UUID
    ) -> SubrecipientRiskScore:
        """
        Factors (weighted):
        - Prior audit findings (30%): material weakness=high, significant deficiency=medium
        - SAM.gov registration status (15%): expired=high, expiring_soon=medium
        - Subaward amount relative to org revenue (15%): >50%=high
        - New vs returning subrecipient (10%): new=higher risk
        - Reporting timeliness history (15%): late reports=higher risk
        - Federal expenditure level (15%): triggers single audit if >= $1M

        Output: numeric score (0-100), risk_level (low/standard/high),
        recommended monitoring plan, key risk factors.
        """
        ...

class SubrecipientRiskScore(BaseModel):
    subrecipient_id: UUID
    score: int  # 0-100
    risk_level: str  # 'low', 'standard', 'high'
    factors: list[RiskFactor]
    monitoring_recommendation: str  # 'annual', 'semi_annual', 'quarterly'
    assessed_at: datetime
```

**Testing**:
- Unit: Subrecipient with material weakness in prior audit scores > 70 (high)
- Unit: New subrecipient with large subaward relative to org size scores medium-high
- Unit: Returning subrecipient with clean audit history and active SAM scores < 30 (low)
- Unit: Monitoring recommendation aligns with risk level

---

## Phase 11: Frontend Dashboard Features

### Purpose
Build out the grantmaker dashboard with full management interfaces for reviews, awards, compliance, and reporting. After this phase, the frontend supports all core workflows.

### Tasks

#### 11.1 — Review Management Interface

**What**: Pages for managing review panels, assigning reviewers, viewing scores, and making decisions.

**Design**:

Pages:
- `/dashboard/reviews` — list review panels with completion status
- `/dashboard/reviews/{panel_id}` — panel detail: assignments grid showing reviewer x application with score status
- `/dashboard/reviews/{panel_id}/scores/{application_id}` — aggregated scores view for a single application
- `/dashboard/my-reviews` — reviewer's personal queue: assigned applications with scoring interface

Scoring interface: form with criteria from rubric, numeric input per criterion (0 to max_score), text comments per criterion, overall comments, recommendation dropdown (fund / fund_with_conditions / decline).

For blind review: application detail view strips organisation name and identifying information.

**Testing**:
- E2E: Programme officer creates panel, assigns reviewers, sees scores appear as reviewers complete reviews
- E2E: Reviewer sees assigned applications, enters scores, submits review
- E2E: Blind review mode hides applicant organisation name
- Accessibility: Scoring form has clear labels and validation messages

#### 11.2 — Award & Financial Management Interface

**What**: Award detail pages with budget tracking, transaction recording, amendment management, and financial charts.

**Design**:

Pages:
- `/dashboard/awards/{id}` — award detail with tabs:
  - Overview: key metrics, status, dates
  - Budget: budget vs. actual table with progress bars
  - Transactions: transaction list with filters and "Add Transaction" form
  - Amendments: amendment history with "New Amendment" flow
  - Compliance: linked compliance obligations
  - Documents: attached files
  - Subrecipients: subrecipient list with risk levels

Budget visualisation: stacked bar chart showing budgeted vs. expended per category, with red highlighting for overspent categories.

**Testing**:
- E2E: Finance officer records expenditure; budget summary updates
- E2E: Creating an amendment and approving it updates award amount/dates
- E2E: Budget chart renders correctly with sample data
- Accessibility: Charts have text alternatives; transaction form is keyboard-navigable

#### 11.3 — Compliance Dashboard

**What**: Cross-award compliance overview with filtering, calendar view of deadlines, and notification management.

**Design**:

Pages:
- `/dashboard/compliance` — tabs:
  - Overview: summary cards (pending, overdue, upcoming 30 days, completed this quarter)
  - Calendar: month view with compliance deadlines marked
  - Table: full obligation list with status filters, award links, and bulk actions
  - Subrecipients: all subrecipients across awards with risk level indicators

Colour coding: overdue (red), due within 7 days (amber), pending (blue), completed (green).

**Testing**:
- E2E: Compliance dashboard shows correct counts
- E2E: Clicking an overdue item navigates to obligation detail
- E2E: Calendar view shows deadlines on correct dates
- Accessibility: Colour-coded status also has text labels and icons

---

## Phase 12: Production Readiness & Deployment

### Purpose
Harden the application for production: containerised deployment, environment configuration, rate limiting, comprehensive logging, performance tuning, and deployment documentation.

### Tasks

#### 12.1 — Production Docker Configuration

**What**: Multi-stage Dockerfile, production Docker Compose, and Nginx reverse proxy configuration.

**Design**:

```dockerfile
# Dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml ./
RUN pip install uv && uv pip install --system -r pyproject.toml

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY backend/ backend/
COPY migrations/ migrations/
COPY alembic.ini .
CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

Production compose adds: Nginx reverse proxy with SSL termination, PostgreSQL with persistent storage and backup cron, Redis with persistence, health checks on all services.

**Testing**:
- Integration: `docker build` succeeds; container starts and responds to health check
- Integration: Nginx proxies API requests correctly
- Integration: Database migrations run on container startup

#### 12.2 — Rate Limiting & Security Hardening

**What**: API rate limiting, CORS hardening, security headers, input sanitisation.

**Design**:

- Rate limiting via `slowapi`: 100 req/min for authenticated users, 20 req/min for unauthenticated
- Security headers middleware: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security`, `Content-Security-Policy`
- Input sanitisation: strip HTML from text fields, validate file uploads, enforce max request body size (10 MB)
- CORS: restrict to configured origins in production

OWASP ASVS alignment: input validation, authentication strength, session management, output encoding.

**Testing**:
- Integration: Exceeding rate limit returns 429 with `Retry-After` header
- Integration: Security headers present on all responses
- Unit: HTML tags are stripped from text input fields
- Integration: CORS blocks requests from unlisted origins

#### 12.3 — Logging, Monitoring & Error Handling

**What**: Structured JSON logging, error tracking, request tracing, and performance metrics.

**Design**:

- Structured logging with `structlog`: JSON format, request ID correlation, user/tenant context
- Error handling: global exception handlers return consistent error responses
- Request tracing: UUID `X-Request-ID` header added to all responses
- Performance: slow query logging (> 500ms), endpoint response time tracking

```python
class ErrorResponse(BaseModel):
    error: str
    detail: str | None = None
    request_id: str
    timestamp: datetime
```

**Testing**:
- Integration: 404 returns `ErrorResponse` with request_id
- Integration: 500 returns `ErrorResponse` without exposing internal details
- Integration: Structured log entries include tenant_id, user_id, and request_id
- Unit: Slow query threshold triggers log entry

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (6 tasks)
    |
Phase 2: Core Grant Lifecycle (5 tasks) ─── requires Phase 1
    |
Phase 3: Review & Award (4 tasks) ─── requires Phase 2
    |
    ├── Phase 4: Financial Management (2 tasks) ─── requires Phase 3
    │       |
    │       └── Phase 5: Compliance & Federal Reporting (4 tasks) ─── requires Phase 4
    │
    ├── Phase 6: Frontend Foundation (3 tasks) ─── requires Phase 2 API
    │       |
    │       └── Phase 11: Dashboard Features (3 tasks) ─── requires Phase 6 + Phases 3-5 APIs
    │
    ├── Phase 7: Federal API Integrations (3 tasks) ─── requires Phase 2 (org/opportunity models)
    │       |
    │       └── Phase 9: Grant Prospecting (3 tasks) ─── requires Phase 7
    │
    └── Phase 8: Reporting & Analytics (2 tasks) ─── requires Phases 3-5

Phase 10: AI-Native Features (4 tasks) ─── requires Phases 2-5 (data), Phase 7 (optional for prospecting)

Phase 12: Production Readiness (3 tasks) ─── can start after Phase 6, completed last
```

**Parallelism opportunities:**
- Phases 4, 6, and 7 can be developed concurrently after Phase 3
- Phases 8, 9, and 10 can be developed concurrently after their prerequisites
- Phase 12 can be developed incrementally alongside any phase after Phase 6

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specifications.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. Ruff linting passes with zero errors (`ruff check .`).
5. Ruff formatting passes (`ruff format --check .`).
6. mypy type checking passes with zero errors (`mypy backend/`).
7. Docker build succeeds (`docker build .`).
8. All new database tables have corresponding Alembic migrations.
9. Alembic upgrade/downgrade cycle is clean (`alembic upgrade head && alembic downgrade -1 && alembic upgrade head`).
10. All new API endpoints appear in the auto-generated OpenAPI spec at `/api/v1/openapi.json`.
11. All new API endpoints have Pydantic request/response schemas with field validation.
12. Audit logging covers all state-changing operations introduced in the phase.
13. New configuration options documented in `.env.example`.
14. E2E tests pass for user-facing features (where applicable).
15. WCAG 2.2 Level AA compliance verified for new frontend pages (where applicable).
