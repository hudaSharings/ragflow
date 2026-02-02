# RAGFlow Backend Migration to Node.js - Comprehensive Plan

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current Architecture Analysis](#2-current-architecture-analysis)
   - 2.1 [Backend Structure](#21-backend-structure)
   - 2.2 [Admin Features Overview](#22-admin-features-overview)
   - 2.3 [User Management Features](#23-user-management-features)
   - 2.4 [Database Schema](#24-database-schema)
   - 2.5 [Authentication & Authorization](#25-authentication--authorization)
3. [Node.js Technology Stack](#3-nodejs-technology-stack)
   - 3.1 [Core Framework & Libraries](#31-core-framework--libraries)
   - 3.2 [Database & ORM](#32-database--orm)
   - 3.3 [Authentication & Security](#33-authentication--security)
   - 3.4 [Additional Tools](#34-additional-tools)
4. [Migration Strategy - Phase by Phase](#4-migration-strategy---phase-by-phase)
   - 4.1 [Phase 1: Foundation & Admin Authentication](#41-phase-1-foundation--admin-authentication)
   - 4.2 [Phase 2: Admin User Management](#42-phase-2-admin-user-management)
   - 4.3 [Phase 3: Admin Service Management](#43-phase-3-admin-service-management)
   - 4.4 [Phase 4: Admin RBAC & Configuration](#44-phase-4-admin-rbac--configuration)
   - 4.5 [Phase 5: User Authentication & Registration](#45-phase-5-user-authentication--registration)
   - 4.6 [Phase 6: User Profile & Settings](#46-phase-6-user-profile--settings)
   - 4.7 [Phase 7: Password Reset & OAuth](#47-phase-7-password-reset--oauth)
   - 4.8 [Phase 8: Tenant Management](#48-phase-8-tenant-management)
   - 4.9 [Phase 9: System Features](#49-phase-9-system-features)
5. [Detailed Implementation Plan - Admin Features](#5-detailed-implementation-plan---admin-features)
   - 5.1 [Project Structure](#51-project-structure)
   - 5.2 [Admin Authentication Module](#52-admin-authentication-module)
   - 5.3 [Admin User Management Module](#53-admin-user-management-module)
   - 5.4 [Admin Service Management Module](#54-admin-service-management-module)
   - 5.5 [Admin RBAC Module](#55-admin-rbac-module)
   - 5.6 [Admin Configuration Module](#56-admin-configuration-module)
6. [Database Migration Strategy](#6-database-migration-strategy)
   - 6.1 [Schema Compatibility](#61-schema-compatibility)
   - 6.2 [Data Access Layer](#62-data-access-layer)
   - 6.3 [Migration Scripts](#63-migration-scripts)
7. [Integration with Existing Python Backend](#7-integration-with-existing-python-backend)
   - 7.1 [Coexistence Strategy](#71-coexistence-strategy)
   - 7.2 [Shared Resources](#72-shared-resources)
   - 7.3 [API Gateway Approach](#73-api-gateway-approach)
8. [Testing Strategy](#8-testing-strategy)
   - 8.1 [Unit Testing](#81-unit-testing)
   - 8.2 [Integration Testing](#82-integration-testing)
   - 8.3 [End-to-End Testing](#83-end-to-end-testing)
   - 8.4 [Migration Testing](#84-migration-testing)
9. [Risk Mitigation](#9-risk-mitigation)
10. [Success Criteria](#10-success-criteria)
11. [Timeline & Milestones](#11-timeline--milestones)
12. [Key Files Reference](#12-key-files-reference)

---

## 1. Executive Summary

This document provides a comprehensive plan for migrating RAGFlow's backend from Python (Flask/Quart) to Node.js, starting with admin features and user management modules. The migration will be executed in phases to minimize disruption and ensure system stability.

**Key Objectives:**
- Migrate admin features (30+ endpoints) to Node.js
- Migrate user management features (15+ endpoints) to Node.js
- Maintain backward compatibility with existing Python services
- Preserve all existing functionality and data integrity
- Implement modern Node.js best practices and architecture

**Migration Approach:**
- **Incremental Migration**: Phase-by-phase approach starting with admin features
- **Coexistence**: Node.js and Python services run in parallel during transition
- **Shared Database**: Both services access the same MySQL database
- **API Gateway**: Route requests to appropriate service (Node.js or Python)

**Estimated Timeline**: 24-28 weeks across 9 phases

---

## 2. Current Architecture Analysis

### 2.1 Backend Structure

**Current Python Backend:**
```
api/
├── apps/              # API Blueprints/Controllers
│   ├── user_app.py    # User management endpoints
│   ├── tenant_app.py  # Tenant management
│   ├── system_app.py  # System features
│   └── ...
├── db/
│   ├── db_models.py   # Database models (Peewee ORM)
│   └── services/      # Business logic layer
│       ├── user_service.py
│       └── ...
└── utils/             # Utility functions

admin/
└── server/            # Admin server (separate Flask app)
    ├── admin_server.py
    ├── routes.py      # Admin endpoints
    ├── services.py    # Admin business logic
    ├── auth.py        # Admin authentication
    └── roles.py       # RBAC (not fully implemented)
```

**Key Technologies:**
- **Framework**: Flask (admin), Quart (main API)
- **ORM**: Peewee
- **Database**: MySQL
- **Authentication**: Flask-Login, JWT tokens
- **Session**: Flask-Session, Redis

### 2.2 Admin Features Overview

**Admin Server** (`admin/server/`):
- Runs on separate port: **9381**
- Separate authentication from regular users
- Requires `is_superuser` flag for access

**Admin Feature Categories:**

1. **Admin Authentication** (4 endpoints)
   - Login, logout, auth verification, ping

2. **User Management** (12 endpoints)
   - CRUD operations, activation, admin privileges, API keys, resource viewing

3. **Service Management** (4 endpoints)
   - Service discovery, health monitoring, restart/shutdown (partially implemented)

4. **RBAC** (9 endpoints)
   - Role CRUD, permission management, user-role assignment (not fully implemented)

5. **System Configuration** (5 endpoints)
   - Variables, configs, environments, version

**Total Admin Endpoints**: ~34 endpoints

### 2.3 User Management Features

**Regular User API** (`api/apps/user_app.py`):

1. **Authentication** (8 endpoints)
   - Login, logout, registration
   - OAuth/OIDC (GitHub, Feishu, configurable)
   - Login channels discovery

2. **User Profile** (2 endpoints)
   - Get profile, update settings

3. **Password Reset** (4 endpoints)
   - Captcha generation, OTP sending, OTP verification, password reset

4. **Tenant Info** (2 endpoints)
   - Get tenant info, update tenant settings

**Total User Endpoints**: ~16 endpoints

### 2.4 Database Schema

**Core Tables:**

1. **`user`** - User accounts
   - `id` (PK, VARCHAR(32))
   - `email` (VARCHAR(255), unique, indexed)
   - `password` (VARCHAR(255), hashed)
   - `nickname` (VARCHAR(100))
   - `access_token` (VARCHAR(255), indexed)
   - `is_superuser` (BOOLEAN)
   - `is_active` (CHAR(1), '0' or '1')
   - `status` (CHAR(1), '0' deleted, '1' active)
   - `login_channel` (VARCHAR)
   - Timestamps: `create_time`, `create_date`, `update_time`, `update_date`

2. **`tenant`** - Tenants/Organizations
   - `id` (PK, VARCHAR(32), same as owner user_id)
   - `name` (VARCHAR)
   - Model IDs: `llm_id`, `embd_id`, `rerank_id`, `asr_id`, `img2txt_id`, `tts_id`
   - `parser_ids` (JSON)
   - `status` (CHAR(1))

3. **`user_tenant`** - User-Tenant relationships
   - `id` (PK, VARCHAR(32))
   - `user_id` (FK → user.id)
   - `tenant_id` (FK → tenant.id)
   - `role` (VARCHAR(32)): 'owner', 'admin', 'normal', 'invite'
   - `invited_by` (VARCHAR(32))
   - `status` (CHAR(1))

4. **`api_token`** - API tokens
   - `id` (PK)
   - `tenant_id` (FK → tenant.id)
   - `token` (VARCHAR, unique)
   - `beta` (VARCHAR(32))

5. **`system_settings`** - System configuration
   - `name` (VARCHAR, unique)
   - `value` (TEXT)
   - `source` (VARCHAR)
   - `data_type` (VARCHAR)

### 2.5 Authentication & Authorization

**Current Authentication Flow:**

1. **Regular Users**:
   - JWT token in `Authorization` header
   - Token stored in `user.access_token`
   - Validated via `@login_required` decorator
   - Session managed by Flask-Login/Quart

2. **Admin Users**:
   - Separate admin server on port 9381
   - Uses Flask-Login for session management
   - `is_superuser` flag required
   - `@check_admin_auth` decorator for authorization

3. **Password Hashing**:
   - Uses `werkzeug.security.generate_password_hash` (pbkdf2:sha256)
   - Password encryption before hashing (via `decrypt()`)

4. **OAuth/OIDC**:
   - Configurable OAuth providers
   - State management via session
   - User creation on first OAuth login

---

## 3. Node.js Technology Stack

### 3.1 Core Framework & Libraries

**Recommended Stack:**

1. **Framework**: **Express.js** or **Fastify**
   - **Express.js**: Most popular, extensive middleware ecosystem
   - **Fastify**: Higher performance, built-in validation
   - **Recommendation**: Express.js for better ecosystem compatibility

2. **TypeScript**: 
   - Type safety
   - Better IDE support
   - Easier refactoring

3. **API Structure**:
   - **Express Router** for route organization
   - **Middleware pattern** for authentication/authorization

### 3.2 Database & ORM

**Options:**

1. **Sequelize** (Recommended)
   - Mature ORM with TypeScript support
   - Good MySQL support
   - Migration tools
   - Associations support

2. **TypeORM**
   - Decorator-based
   - Active Record and Data Mapper patterns
   - Good TypeScript integration

3. **Prisma**
   - Modern ORM
   - Type-safe database client
   - Excellent migration system
   - **Recommendation**: Consider for new projects, but Sequelize for compatibility

**Recommendation**: **Sequelize** for better compatibility with existing schema

### 3.3 Authentication & Security

1. **JWT**: **jsonwebtoken** + **@types/jsonwebtoken**
2. **Password Hashing**: **bcrypt** or **argon2**
   - **Note**: Must match Python's `werkzeug.security` format for compatibility
   - May need custom implementation or compatibility layer
3. **Session Management**: **express-session** + **connect-redis**
4. **OAuth/OIDC**: **passport** + **passport-oauth2**, **passport-github2**
5. **Validation**: **joi** or **express-validator**
6. **Encryption**: **crypto** (built-in) for password decryption compatibility

### 3.4 Additional Tools

1. **Logging**: **winston** or **pino**
2. **Configuration**: **dotenv** + **config**
3. **Email**: **nodemailer**
4. **Redis Client**: **ioredis**
5. **HTTP Client**: **axios** or **node-fetch**
6. **Testing**: **Jest** + **Supertest**
7. **Code Quality**: **ESLint** + **Prettier**
8. **API Documentation**: **swagger-jsdoc** + **swagger-ui-express**

---

## 4. Migration Strategy - Phase by Phase

### 4.1 Phase 1: Foundation & Admin Authentication

**Duration**: 2-3 weeks

**Objectives:**
- Set up Node.js project structure
- Configure database connection
- Implement admin authentication endpoints
- Set up shared authentication middleware

**Deliverables:**
- [ ] Node.js project initialized with TypeScript
- [ ] Database connection and Sequelize models for `user` table
- [ ] Admin login endpoint (`POST /api/v1/admin/login`)
- [ ] Admin logout endpoint (`GET /api/v1/admin/logout`)
- [ ] Admin auth verification (`GET /api/v1/admin/auth`)
- [ ] Admin ping endpoint (`GET /api/v1/admin/ping`)
- [ ] Admin authentication middleware
- [ ] Password hashing compatibility layer
- [ ] JWT token generation/validation
- [ ] Session management setup

**Key Files to Create:**
```
backend-nodejs/
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   └── config.ts
│   ├── models/
│   │   └── User.ts
│   ├── middleware/
│   │   ├── auth.middleware.ts
│   │   └── adminAuth.middleware.ts
│   ├── services/
│   │   ├── auth.service.ts
│   │   └── user.service.ts
│   ├── controllers/
│   │   └── admin/
│   │       └── auth.controller.ts
│   ├── routes/
│   │   └── admin/
│   │       └── auth.routes.ts
│   └── utils/
│       ├── password.util.ts
│       └── jwt.util.ts
├── package.json
└── tsconfig.json
```

**Implementation Notes:**
- Password hashing must be compatible with Python's `werkzeug.security`
- May need to implement custom password hasher or use compatibility library
- JWT tokens should match Python's format for seamless integration

### 4.2 Phase 2: Admin User Management

**Duration**: 3-4 weeks

**Objectives:**
- Implement all admin user management endpoints
- User CRUD operations
- User activation/deactivation
- Admin privilege management
- API key management for users

**Deliverables:**
- [ ] List all users (`GET /api/v1/admin/users`)
- [ ] Create user (`POST /api/v1/admin/users`)
- [ ] Get user details (`GET /api/v1/admin/users/:username`)
- [ ] Delete user (`DELETE /api/v1/admin/users/:username`)
- [ ] Change user password (`PUT /api/v1/admin/users/:username/password`)
- [ ] Activate/deactivate user (`PUT /api/v1/admin/users/:username/activate`)
- [ ] Grant admin (`PUT /api/v1/admin/users/:username/admin`)
- [ ] Revoke admin (`DELETE /api/v1/admin/users/:username/admin`)
- [ ] Get user datasets (`GET /api/v1/admin/users/:username/datasets`)
- [ ] Get user agents (`GET /api/v1/admin/users/:username/agents`)
- [ ] Generate API key (`POST /api/v1/admin/users/:username/keys`)
- [ ] List API keys (`GET /api/v1/admin/users/:username/keys`)
- [ ] Delete API key (`DELETE /api/v1/admin/users/:username/keys/:key`)

**Key Files to Create:**
```
src/
├── models/
│   ├── Tenant.ts
│   ├── UserTenant.ts
│   └── ApiToken.ts
├── services/
│   ├── admin/
│   │   └── userManagement.service.ts
│   └── knowledgebase.service.ts (for datasets)
├── controllers/
│   └── admin/
│       └── user.controller.ts
└── routes/
    └── admin/
        └── user.routes.ts
```

**Implementation Notes:**
- User deletion should be soft delete (set `status = '0'`)
- Email validation required
- Password encryption/decryption compatibility needed
- Integration with knowledgebase and agent services (may call Python APIs initially)

### 4.3 Phase 3: Admin Service Management

**Duration**: 2 weeks

**Objectives:**
- Implement service discovery and monitoring
- Service health checks
- Service management endpoints

**Deliverables:**
- [ ] List all services (`GET /api/v1/admin/services`)
- [ ] Get services by type (`GET /api/v1/admin/service_types/:type`)
- [ ] Get service details (`GET /api/v1/admin/services/:id`)
- [ ] Shutdown service (`DELETE /api/v1/admin/services/:id`) - if implemented
- [ ] Restart service (`PUT /api/v1/admin/services/:id`) - if implemented

**Key Files to Create:**
```
src/
├── services/
│   └── admin/
│       └── serviceManagement.service.ts
├── controllers/
│   └── admin/
│       └── service.controller.ts
└── routes/
    └── admin/
        └── service.routes.ts
```

**Implementation Notes:**
- Service configuration may be in config files or environment variables
- Health check utilities need to be ported or integrated
- Service restart/shutdown may not be fully implemented in Python version

### 4.4 Phase 4: Admin RBAC & Configuration

**Duration**: 3-4 weeks

**Objectives:**
- Implement RBAC system (if required)
- System configuration management
- Environment variable management

**Deliverables:**
- [ ] Role CRUD operations
- [ ] Permission management
- [ ] User-role assignment
- [ ] System variables management
- [ ] Configuration retrieval
- [ ] Environment variables listing
- [ ] Version endpoint

**Key Files to Create:**
```
src/
├── models/
│   ├── Role.ts
│   ├── Permission.ts
│   └── SystemSetting.ts
├── services/
│   └── admin/
│       ├── rbac.service.ts
│       └── config.service.ts
└── controllers/
    └── admin/
        ├── role.controller.ts
        └── config.controller.ts
```

**Implementation Notes:**
- RBAC may not be fully implemented in Python version
- Can implement basic structure for future expansion
- System settings stored in database table

### 4.5 Phase 5: User Authentication & Registration

**Duration**: 3 weeks

**Objectives:**
- Implement regular user authentication
- User registration
- OAuth/OIDC integration

**Deliverables:**
- [ ] User login (`POST /user/login`)
- [ ] User logout (`GET /user/logout`)
- [ ] User registration (`POST /user/register`)
- [ ] Get login channels (`GET /user/login/channels`)
- [ ] OAuth login redirect (`GET /user/login/:channel`)
- [ ] OAuth callback (`GET /user/oauth/callback/:channel`)
- [ ] User profile retrieval (`GET /user/info`)

**Key Files to Create:**
```
src/
├── controllers/
│   └── user/
│       └── auth.controller.ts
├── services/
│   └── user/
│       ├── auth.service.ts
│       └── oauth.service.ts
└── routes/
    └── user/
        └── auth.routes.ts
```

**Implementation Notes:**
- OAuth providers are configurable
- Need to handle state management for OAuth
- User creation on first OAuth login
- Avatar download from OAuth providers

### 4.6 Phase 6: User Profile & Settings

**Duration**: 2 weeks

**Objectives:**
- User profile management
- User settings update
- Tenant information management

**Deliverables:**
- [ ] Update user settings (`POST /user/setting`)
- [ ] Get tenant info (`GET /user/tenant_info`)
- [ ] Update tenant info (`POST /user/set_tenant_info`)

**Key Files to Create:**
```
src/
├── controllers/
│   └── user/
│       └── profile.controller.ts
└── services/
    └── user/
        └── profile.service.ts
```

### 4.7 Phase 7: Password Reset & OAuth

**Duration**: 3 weeks

**Objectives:**
- Password reset flow
- Captcha generation
- OTP management
- Email sending

**Deliverables:**
- [ ] Generate captcha (`GET /user/forget/captcha`)
- [ ] Send OTP (`POST /user/forget/otp`)
- [ ] Verify OTP (`POST /user/forget/verify-otp`)
- [ ] Reset password (`POST /user/forget/reset-password`)

**Key Files to Create:**
```
src/
├── services/
│   └── user/
│       ├── passwordReset.service.ts
│       └── captcha.service.ts
├── utils/
│   └── email.util.ts
└── controllers/
    └── user/
        └── passwordReset.controller.ts
```

**Implementation Notes:**
- Captcha generation using `captcha` library
- OTP stored in Redis with TTL
- Email templates need to be ported
- Rate limiting for OTP requests

### 4.8 Phase 8: Tenant Management

**Duration**: 2-3 weeks

**Objectives:**
- Tenant CRUD operations
- User-tenant relationships
- Tenant invitations

**Deliverables:**
- [ ] List tenants (`GET /tenant/list`)
- [ ] List tenant users (`GET /tenant/:tenant_id/user/list`)
- [ ] Invite user to tenant (`POST /tenant/:tenant_id/user`)
- [ ] Remove user from tenant (`DELETE /tenant/:tenant_id/user/:user_id`)
- [ ] Accept invitation (`PUT /tenant/agree/:tenant_id`)

**Key Files to Create:**
```
src/
├── controllers/
│   └── tenant/
│       └── tenant.controller.ts
└── services/
    └── tenant/
        └── tenant.service.ts
```

### 4.9 Phase 9: System Features

**Duration**: 2 weeks

**Objectives:**
- System status endpoints
- API token management
- System configuration

**Deliverables:**
- [ ] System status (`GET /system/status`)
- [ ] Health check (`GET /system/healthz`)
- [ ] Ping (`GET /system/ping`)
- [ ] Version (`GET /system/version`)
- [ ] Generate API token (`POST /system/new_token`)
- [ ] List API tokens (`GET /system/token_list`)
- [ ] Delete API token (`DELETE /system/token/:token`)
- [ ] Get config (`GET /system/config`)

**Key Files to Create:**
```
src/
├── controllers/
│   └── system/
│       └── system.controller.ts
└── services/
    └── system/
        └── system.service.ts
```

---

## 5. Detailed Implementation Plan - Admin Features

### 5.1 Project Structure

**Recommended Structure:**

```
backend-nodejs/
├── src/
│   ├── config/                 # Configuration files
│   │   ├── database.ts         # Database connection
│   │   ├── redis.ts            # Redis connection
│   │   ├── config.ts           # App configuration
│   │   └── env.ts              # Environment variables
│   │
│   ├── models/                 # Sequelize models
│   │   ├── index.ts            # Model initialization
│   │   ├── User.ts
│   │   ├── Tenant.ts
│   │   ├── UserTenant.ts
│   │   ├── ApiToken.ts
│   │   └── SystemSetting.ts
│   │
│   ├── middleware/             # Express middleware
│   │   ├── auth.middleware.ts  # JWT authentication
│   │   ├── adminAuth.middleware.ts
│   │   ├── error.middleware.ts
│   │   └── validation.middleware.ts
│   │
│   ├── services/               # Business logic
│   │   ├── admin/
│   │   │   ├── userManagement.service.ts
│   │   │   ├── serviceManagement.service.ts
│   │   │   ├── rbac.service.ts
│   │   │   └── config.service.ts
│   │   ├── user/
│   │   │   ├── auth.service.ts
│   │   │   ├── profile.service.ts
│   │   │   └── passwordReset.service.ts
│   │   └── tenant/
│   │       └── tenant.service.ts
│   │
│   ├── controllers/            # Request handlers
│   │   ├── admin/
│   │   │   ├── auth.controller.ts
│   │   │   ├── user.controller.ts
│   │   │   ├── service.controller.ts
│   │   │   ├── role.controller.ts
│   │   │   └── config.controller.ts
│   │   ├── user/
│   │   │   ├── auth.controller.ts
│   │   │   ├── profile.controller.ts
│   │   │   └── passwordReset.controller.ts
│   │   └── tenant/
│   │       └── tenant.controller.ts
│   │
│   ├── routes/                  # Route definitions
│   │   ├── admin/
│   │   │   ├── index.ts
│   │   │   ├── auth.routes.ts
│   │   │   ├── user.routes.ts
│   │   │   ├── service.routes.ts
│   │   │   ├── role.routes.ts
│   │   │   └── config.routes.ts
│   │   ├── user/
│   │   │   ├── index.ts
│   │   │   ├── auth.routes.ts
│   │   │   ├── profile.routes.ts
│   │   │   └── passwordReset.routes.ts
│   │   └── tenant/
│   │       └── tenant.routes.ts
│   │
│   ├── utils/                   # Utility functions
│   │   ├── password.util.ts    # Password hashing/compatibility
│   │   ├── jwt.util.ts         # JWT token management
│   │   ├── crypt.util.ts       # Encryption/decryption
│   │   ├── email.util.ts       # Email sending
│   │   ├── captcha.util.ts     # Captcha generation
│   │   └── response.util.ts    # Response formatting
│   │
│   ├── types/                   # TypeScript types
│   │   ├── user.types.ts
│   │   ├── admin.types.ts
│   │   └── common.types.ts
│   │
│   ├── validators/              # Request validation
│   │   ├── user.validator.ts
│   │   └── admin.validator.ts
│   │
│   ├── errors/                  # Custom error classes
│   │   ├── AppError.ts
│   │   └── errorHandler.ts
│   │
│   └── app.ts                   # Express app setup
│
├── tests/                       # Test files
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── migrations/                  # Database migrations
├── scripts/                     # Utility scripts
├── .env.example
├── .gitignore
├── package.json
├── tsconfig.json
└── README.md
```

### 5.2 Admin Authentication Module

**Key Components:**

1. **Admin Auth Controller** (`controllers/admin/auth.controller.ts`):
```typescript
// Pseudo-code structure
export class AdminAuthController {
  async login(req, res, next) {
    // Validate email/password
    // Check is_superuser flag
    // Generate access_token
    // Create session
    // Return JWT token
  }

  async logout(req, res, next) {
    // Invalidate access_token
    // Clear session
  }

  async verifyAuth(req, res, next) {
    // Verify admin authorization
  }

  async ping(req, res, next) {
    // Health check
  }
}
```

2. **Admin Auth Middleware** (`middleware/adminAuth.middleware.ts`):
```typescript
export const checkAdminAuth = async (req, res, next) => {
  // Verify JWT token
  // Check is_superuser flag
  // Check is_active status
  // Attach user to request
};
```

3. **Password Compatibility** (`utils/password.util.ts`):
```typescript
// Must implement werkzeug.security compatible hashing
// Python uses: pbkdf2:sha256:260000$salt$hash
export function hashPassword(password: string): string {
  // Implement compatible hashing
}

export function checkPassword(password: string, hash: string): boolean {
  // Implement compatible checking
}
```

**Implementation Challenges:**
- Password hashing format compatibility with Python's werkzeug
- Session management compatibility
- JWT token format matching

### 5.3 Admin User Management Module

**Key Components:**

1. **User Management Service** (`services/admin/userManagement.service.ts`):
   - User CRUD operations
   - User activation/deactivation
   - Admin privilege management
   - API key management
   - Resource retrieval (datasets, agents)

2. **User Controller** (`controllers/admin/user.controller.ts`):
   - Route handlers for all user management endpoints
   - Request validation
   - Response formatting

3. **Database Models**:
   - User model with all fields
   - Relationships with Tenant, ApiToken

**Key Operations:**
- Create user with email validation
- Soft delete (status = '0')
- Password encryption/decryption compatibility
- Integration with knowledgebase service for datasets
- Integration with agent service for agents

### 5.4 Admin Service Management Module

**Key Components:**

1. **Service Management Service** (`services/admin/serviceManagement.service.ts`):
   - Service discovery from configuration
   - Health check execution
   - Service status monitoring

2. **Service Controller** (`controllers/admin/service.controller.ts`):
   - List services
   - Get service details
   - Service management operations

**Implementation Notes:**
- Service configuration may be in files or environment
- Health check utilities need porting
- May need to call Python health check functions initially

### 5.5 Admin RBAC Module

**Key Components:**

1. **RBAC Service** (`services/admin/rbac.service.ts`):
   - Role CRUD
   - Permission management
   - User-role assignment

2. **Role Controller** (`controllers/admin/role.controller.ts`):
   - Role management endpoints
   - Permission endpoints

**Implementation Notes:**
- RBAC may not be fully implemented in Python
- Can implement structure for future use
- Database schema may need to be designed

### 5.6 Admin Configuration Module

**Key Components:**

1. **Config Service** (`services/admin/config.service.ts`):
   - System settings management
   - Configuration retrieval
   - Environment variable access

2. **Config Controller** (`controllers/admin/config.controller.ts`):
   - Configuration endpoints
   - Version endpoint

---

## 6. Database Migration Strategy

### 6.1 Schema Compatibility

**Approach:**
- Use existing database schema (no schema changes)
- Sequelize models match existing table structure
- Maintain field types and constraints

**Key Considerations:**
- VARCHAR lengths
- CHAR(1) for status flags
- JSON fields (parser_ids)
- Timestamp formats

### 6.2 Data Access Layer

**Sequelize Models:**

```typescript
// Example User model
@Table({ tableName: 'user', timestamps: false })
export class User extends Model {
  @PrimaryKey
  @Column({ type: DataType.STRING(32) })
  id: string;

  @Column({ type: DataType.STRING(255), allowNull: true })
  access_token: string;

  @Column({ type: DataType.STRING(100), allowNull: false })
  nickname: string;

  @Column({ type: DataType.STRING(255), allowNull: true })
  password: string;

  @Column({ type: DataType.STRING(255), allowNull: false, unique: true })
  email: string;

  @Column({ type: DataType.BOOLEAN, allowNull: true, defaultValue: false })
  is_superuser: boolean;

  @Column({ type: DataType.CHAR(1), allowNull: false, defaultValue: '1' })
  is_active: string;

  @Column({ type: DataType.CHAR(1), allowNull: true, defaultValue: '1' })
  status: string;

  // ... other fields
}
```

### 6.3 Migration Scripts

**Initial Setup:**
- No data migration needed (using existing database)
- May need to add indexes if missing
- Verify data integrity

**Future Migrations:**
- Use Sequelize migrations for schema changes
- Coordinate with Python backend for schema updates

---

## 7. Integration with Existing Python Backend

### 7.1 Coexistence Strategy

**Phase 1: Parallel Running**
- Node.js admin server runs alongside Python admin server
- Different ports or routing via API gateway
- Both access same database

**Phase 2: Gradual Migration**
- Route specific endpoints to Node.js
- Keep Python endpoints as fallback
- Monitor and compare responses

**Phase 3: Full Migration**
- All admin endpoints on Node.js
- Python admin server deprecated
- Python main API still running

### 7.2 Shared Resources

**Database:**
- Both services read/write to same MySQL database
- Use transactions carefully to avoid conflicts
- Coordinate schema changes

**Redis:**
- Shared session storage
- Shared cache
- OTP storage for password reset

**File Storage:**
- Shared MinIO/S3 storage
- Avatar images
- Document storage

### 7.3 API Gateway Approach

**Option 1: Nginx/Reverse Proxy**
```
Client → Nginx → {
  /api/v1/admin/* → Node.js (port 9381)
  /user/* → Node.js (port 9380)
  /* → Python (port 9380)
}
```

**Option 2: API Gateway Service**
- Kong, AWS API Gateway, or custom gateway
- Route based on path patterns
- Load balancing and health checks

**Option 3: Service Mesh**
- Istio, Linkerd (for microservices architecture)
- More complex but provides advanced features

---

## 8. Testing Strategy

### 8.1 Unit Testing

**Tools:** Jest

**Coverage:**
- Service layer functions
- Utility functions
- Middleware functions
- Password hashing/compatibility
- JWT token generation/validation

**Example:**
```typescript
describe('AdminAuthService', () => {
  it('should authenticate admin user', async () => {
    // Test admin login
  });

  it('should reject non-admin user', async () => {
    // Test admin login with regular user
  });
});
```

### 8.2 Integration Testing

**Tools:** Jest + Supertest

**Coverage:**
- API endpoints
- Database operations
- Redis operations
- External service calls

**Example:**
```typescript
describe('POST /api/v1/admin/login', () => {
  it('should return JWT token on successful login', async () => {
    const response = await request(app)
      .post('/api/v1/admin/login')
      .send({ email: 'admin@ragflow.io', password: 'admin' });
    
    expect(response.status).toBe(200);
    expect(response.body.data).toHaveProperty('token');
  });
});
```

### 8.3 End-to-End Testing

**Tools:** Jest + Supertest or Playwright

**Coverage:**
- Complete user flows
- Admin workflows
- OAuth flows
- Password reset flows

### 8.4 Migration Testing

**Objectives:**
- Verify Node.js responses match Python responses
- Data consistency checks
- Performance comparison
- Error handling compatibility

**Approach:**
- Run both services in parallel
- Compare responses for same requests
- Monitor database state
- Load testing

---

## 9. Risk Mitigation

### 9.1 Technical Risks

**Risk 1: Password Hashing Incompatibility**
- **Mitigation**: Implement compatibility layer, test thoroughly
- **Fallback**: Keep Python password hashing service initially

**Risk 2: Session Management Differences**
- **Mitigation**: Use same Redis session store, compatible format
- **Fallback**: Shared session storage approach

**Risk 3: Database Connection Issues**
- **Mitigation**: Connection pooling, retry logic, monitoring
- **Fallback**: Failover to Python service

**Risk 4: OAuth Integration Complexity**
- **Mitigation**: Thorough testing with all providers
- **Fallback**: Keep Python OAuth handlers initially

### 9.2 Operational Risks

**Risk 1: Service Downtime During Migration**
- **Mitigation**: Gradual migration, parallel running, rollback plan
- **Fallback**: Route traffic back to Python service

**Risk 2: Data Inconsistency**
- **Mitigation**: Transaction management, data validation, monitoring
- **Fallback**: Database backup and restore procedures

**Risk 3: Performance Degradation**
- **Mitigation**: Performance testing, optimization, monitoring
- **Fallback**: Scale horizontally, optimize queries

### 9.3 Business Risks

**Risk 1: Feature Gaps**
- **Mitigation**: Comprehensive feature comparison, testing
- **Fallback**: Keep Python service for missing features

**Risk 2: User Experience Impact**
- **Mitigation**: Maintain API compatibility, thorough testing
- **Fallback**: Quick rollback capability

---

## 10. Success Criteria

### 10.1 Functional Criteria

- [ ] All admin endpoints implemented and tested
- [ ] All user management endpoints implemented and tested
- [ ] Password reset flow working
- [ ] OAuth integration working
- [ ] Tenant management working
- [ ] System features working

### 10.2 Performance Criteria

- [ ] Response times within 200ms for 95% of requests
- [ ] Database query performance equal or better than Python
- [ ] Memory usage within acceptable limits
- [ ] CPU usage optimized

### 10.3 Quality Criteria

- [ ] Test coverage > 80%
- [ ] Zero critical bugs
- [ ] API response format matches Python version
- [ ] Error handling comprehensive
- [ ] Logging and monitoring in place

### 10.4 Compatibility Criteria

- [ ] Database schema compatibility maintained
- [ ] Password hashing compatibility verified
- [ ] JWT token format compatible
- [ ] Session management compatible
- [ ] Frontend integration seamless

---

## 11. Timeline & Milestones

**Total Duration: 24-28 weeks**

| Phase | Duration | Milestones |
|-------|----------|------------|
| **Phase 1** | 2-3 weeks | Foundation & Admin Auth |
| **Phase 2** | 3-4 weeks | Admin User Management |
| **Phase 3** | 2 weeks | Admin Service Management |
| **Phase 4** | 3-4 weeks | Admin RBAC & Config |
| **Phase 5** | 3 weeks | User Auth & Registration |
| **Phase 6** | 2 weeks | User Profile & Settings |
| **Phase 7** | 3 weeks | Password Reset & OAuth |
| **Phase 8** | 2-3 weeks | Tenant Management |
| **Phase 9** | 2 weeks | System Features |
| **Testing & Polish** | 2-3 weeks | Comprehensive testing, bug fixes |

**Key Milestones:**
- **Week 3**: Admin authentication working
- **Week 7**: Admin user management complete
- **Week 12**: All admin features complete
- **Week 15**: User authentication complete
- **Week 20**: All user features complete
- **Week 24**: Full migration complete, testing done
- **Week 28**: Production deployment ready

---

## 12. Key Files Reference

### Python Backend Files (Reference)

**Admin Server:**
- `admin/server/admin_server.py` - Admin server entry point
- `admin/server/routes.py` - Admin route definitions
- `admin/server/services.py` - Admin business logic
- `admin/server/auth.py` - Admin authentication
- `admin/server/roles.py` - RBAC (not fully implemented)

**Main API:**
- `api/apps/user_app.py` - User management endpoints
- `api/apps/tenant_app.py` - Tenant management
- `api/apps/system_app.py` - System features

**Database:**
- `api/db/db_models.py` - Database models
- `api/db/services/user_service.py` - User service
- `api/db/services/api_service.py` - API token service

**Utilities:**
- `api/utils/crypt.py` - Encryption/decryption
- `api/utils/web_utils.py` - Web utilities (OTP, captcha, email)
- `api/utils/api_utils.py` - API utilities

### Node.js Files to Create

**Core:**
- `src/app.ts` - Express app setup
- `src/config/database.ts` - Database configuration
- `src/config/config.ts` - App configuration

**Models:**
- `src/models/User.ts`
- `src/models/Tenant.ts`
- `src/models/UserTenant.ts`
- `src/models/ApiToken.ts`
- `src/models/SystemSetting.ts`

**Admin:**
- `src/controllers/admin/auth.controller.ts`
- `src/controllers/admin/user.controller.ts`
- `src/controllers/admin/service.controller.ts`
- `src/controllers/admin/role.controller.ts`
- `src/controllers/admin/config.controller.ts`

**User:**
- `src/controllers/user/auth.controller.ts`
- `src/controllers/user/profile.controller.ts`
- `src/controllers/user/passwordReset.controller.ts`

**Services:**
- `src/services/admin/userManagement.service.ts`
- `src/services/user/auth.service.ts`
- `src/services/user/passwordReset.service.ts`

**Utilities:**
- `src/utils/password.util.ts` - Password hashing compatibility
- `src/utils/jwt.util.ts` - JWT management
- `src/utils/crypt.util.ts` - Encryption compatibility
- `src/utils/email.util.ts` - Email sending

---

## Appendix A: Password Hashing Compatibility

**Python (werkzeug.security):**
- Format: `pbkdf2:sha256:260000$salt$hash`
- Algorithm: PBKDF2 with SHA256
- Iterations: 260000
- Salt: Random 16 bytes (hex encoded)
- Hash: 32 bytes (hex encoded)

**Node.js Implementation:**
```typescript
import crypto from 'crypto';
import pbkdf2 from 'pbkdf2';

export function hashPassword(password: string): string {
  const salt = crypto.randomBytes(16);
  const iterations = 260000;
  const hash = pbkdf2.pbkdf2Sync(
    password,
    salt,
    iterations,
    32,
    'sha256'
  );
  return `pbkdf2:sha256:${iterations}$${salt.toString('hex')}$${hash.toString('hex')}`;
}

export function checkPassword(password: string, hash: string): boolean {
  const parts = hash.split('$');
  if (parts.length !== 3) return false;
  
  const [method, iterations, saltHash] = parts;
  const [algorithm, hashType, iterCount] = method.split(':');
  
  const salt = Buffer.from(saltHash.split('$')[0], 'hex');
  const storedHash = Buffer.from(saltHash.split('$')[1], 'hex');
  
  const computedHash = pbkdf2.pbkdf2Sync(
    password,
    salt,
    parseInt(iterCount),
    32,
    hashType
  );
  
  return crypto.timingSafeEqual(computedHash, storedHash);
}
```

---

## Appendix B: Environment Variables

**Required Environment Variables:**

```bash
# Database
DB_HOST=localhost
DB_PORT=3306
DB_NAME=ragflow
DB_USER=root
DB_PASSWORD=password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# JWT
JWT_SECRET=your-secret-key
JWT_EXPIRES_IN=7d

# Server
PORT=9381
NODE_ENV=development

# Email (for password reset)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-password

# OAuth
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
FEISHU_APP_ID=
FEISHU_APP_SECRET=

# Encryption (must match Python)
SECRET_KEY=your-secret-key-for-encryption
```

---

## Appendix C: API Response Format

**Standard Response Format:**

```typescript
// Success Response
{
  "code": 0,
  "data": { ... },
  "message": "Success message"
}

// Error Response
{
  "code": 500,
  "data": false,
  "message": "Error message"
}
```

**Response Codes:**
- `0`: Success
- `400`: Bad Request
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Not Found
- `500`: Internal Server Error

---

## Conclusion

This migration plan provides a comprehensive roadmap for migrating RAGFlow's backend from Python to Node.js, starting with admin features and user management. The phased approach ensures minimal disruption and allows for thorough testing at each stage.

**Next Steps:**
1. Review and approve this plan
2. Set up development environment
3. Begin Phase 1 implementation
4. Establish CI/CD pipeline
5. Set up monitoring and logging

**Key Success Factors:**
- Maintain API compatibility
- Thorough testing at each phase
- Gradual migration with rollback capability
- Team collaboration and code reviews
- Documentation and knowledge sharing

---

*Document Version: 1.0*  
*Last Updated: 2025-01-27*  
*Author: AI Assistant*
