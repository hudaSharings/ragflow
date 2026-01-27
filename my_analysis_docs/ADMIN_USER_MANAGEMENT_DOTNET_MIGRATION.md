# Admin & User Management Features - .NET Migration Plan

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Complete Feature Inventory](#1-complete-feature-inventory)
   - 2.1 [Admin Features](#11-admin-features-admin-server---adminserver)
   - 2.2 [User Management Features](#12-user-management-features-regular-api---apiappsuser_apppy)
   - 2.3 [System Features](#13-system-features-regular-api---apiappssystem_apppy)
3. [Database Models & Schema](#2-database-models--schema)
   - 3.1 [Core Tables](#21-core-tables)
   - 3.2 [Database Access Patterns](#22-database-access-patterns)
4. [Dependencies & Integration Points](#3-dependencies--integration-points)
   - 4.1 [How Other Features Depend on User/Admin Features](#31-how-other-features-depend-on-useradmin-features)
   - 4.2 [External Dependencies](#32-external-dependencies)
5. [Migration Strategy for .NET](#4-migration-strategy-for-net)
   - 5.1 [Phase 1: Core User Management](#41-phase-1-core-user-management-net)
   - 5.2 [Phase 2: Password Reset & OAuth](#42-phase-2-password-reset--oauth-net)
   - 5.3 [Phase 3: Tenant Management](#43-phase-3-tenant-management-net)
   - 5.4 [Phase 4: Admin Features](#44-phase-4-admin-features-net)
   - 5.5 [Phase 5: System Features](#45-phase-5-system-features-net)
6. [.NET Implementation Details](#5-net-implementation-details)
   - 6.1 [Technology Stack Recommendations](#51-technology-stack-recommendations)
   - 6.2 [Project Structure](#52-project-structure)
   - 6.3 [Key Implementation Patterns](#53-key-implementation-patterns)
   - 6.4 [Database Migration Strategy](#54-database-migration-strategy)
7. [Integration with Existing Python Code](#6-integration-with-existing-python-code)
   - 7.1 [During Migration](#61-during-migration)
   - 7.2 [Authentication Compatibility](#62-authentication-compatibility)
   - 7.3 [Data Consistency](#63-data-consistency)
8. [Testing Strategy](#7-testing-strategy)
   - 8.1 [Unit Tests](#71-unit-tests)
   - 8.2 [Integration Tests](#72-integration-tests)
   - 8.3 [End-to-End Tests](#73-end-to-end-tests)
   - 8.4 [Compatibility Tests](#74-compatibility-tests)
9. [Risk Mitigation](#8-risk-mitigation)
10. [Success Criteria](#9-success-criteria)
11. [Key Files Reference](#10-key-files-reference)
12. [Next Steps](#11-next-steps)

---

## Executive Summary

This document provides a comprehensive analysis of all admin and user management features in RAGFlow, their dependencies, and how other features depend on them. This is specifically focused on migrating these features to .NET platform.

**Quick Overview**:
- **Admin Features**: 30+ endpoints for user management, service monitoring, RBAC, system configuration
- **User Management**: 15+ endpoints for authentication, profile, tenant management, password reset
- **Migration Timeline**: 22 weeks across 5 phases
- **Dependencies**: All features require authentication; multi-tenancy enforced throughout
- **Integration**: Shared database, JWT tokens, Redis sessions

---

## 1. Complete Feature Inventory

### 1.1 Admin Features (Admin Server - `admin/server/`)

The admin server runs on a separate port (9381) and provides system administration capabilities.

#### **A. Admin Authentication**
- **Location**: `admin/server/auth.py`, `admin/server/routes.py`
- **Endpoints**:
  - `POST /api/v1/admin/login` - Admin login
  - `GET /api/v1/admin/logout` - Admin logout
  - `GET /api/v1/admin/auth` - Verify admin authorization
  - `GET /api/v1/admin/ping` - Health check

**Key Details**:
- Default admin: `admin@ragflow.io` / `admin`
- Uses Flask-Login for session management
- Separate authentication from regular user auth
- `is_superuser` flag in User table determines admin access

#### **B. User Management (Admin)**
- **Location**: `admin/server/services.py::UserMgr`
- **Endpoints**:
  - `GET /api/v1/admin/users` - List all users
  - `POST /api/v1/admin/users` - Create new user
  - `GET /api/v1/admin/users/<username>` - Get user details
  - `DELETE /api/v1/admin/users/<username>` - Delete user
  - `PUT /api/v1/admin/users/<username>/password` - Change user password
  - `PUT /api/v1/admin/users/<username>/activate` - Activate/deactivate user
  - `PUT /api/v1/admin/users/<username>/admin` - Grant admin privileges
  - `DELETE /api/v1/admin/users/<username>/admin` - Revoke admin privileges
  - `GET /api/v1/admin/users/<username>/datasets` - Get user's knowledge bases
  - `GET /api/v1/admin/users/<username>/agents` - Get user's agents
  - `POST /api/v1/admin/users/<username>/keys` - Generate API key for user
  - `GET /api/v1/admin/users/<username>/keys` - List user's API keys
  - `DELETE /api/v1/admin/users/<username>/keys/<key>` - Delete API key

**Operations**:
- Create users with username/password/role
- Delete users (soft delete - sets status to 0)
- Activate/deactivate users
- Grant/revoke admin privileges
- View user's resources (datasets, agents)
- Manage user API keys

#### **C. Service Management**
- **Location**: `admin/server/services.py::ServiceMgr`
- **Endpoints**:
  - `GET /api/v1/admin/services` - List all services
  - `GET /api/v1/admin/service_types/<service_type>` - Get services by type
  - `GET /api/v1/admin/services/<service_id>` - Get service details
  - `DELETE /api/v1/admin/services/<service_id>` - Shutdown service
  - `PUT /api/v1/admin/services/<service_id>` - Restart service

**Monitored Services**:
- RAGFlow server
- Task Executor processes
- MySQL
- Elasticsearch/Infinity
- Redis
- MinIO

#### **D. Role-Based Access Control (RBAC)**
- **Location**: `admin/server/roles.py::RoleMgr`
- **Endpoints**:
  - `POST /api/v1/admin/roles` - Create role
  - `GET /api/v1/admin/roles` - List all roles
  - `PUT /api/v1/admin/roles/<role_name>` - Update role description
  - `DELETE /api/v1/admin/roles/<role_name>` - Delete role
  - `GET /api/v1/admin/roles/<role_name>/permission` - Get role permissions
  - `POST /api/v1/admin/roles/<role_name>/permission` - Grant permission to role
  - `DELETE /api/v1/admin/roles/<role_name>/permission` - Revoke permission from role
  - `PUT /api/v1/admin/users/<user_name>/role` - Assign role to user
  - `GET /api/v1/admin/users/<user_name>/permission` - Get user permissions

**Note**: RBAC appears to be enterprise feature (may not be in all versions)

#### **E. System Configuration**
- **Location**: `admin/server/services.py::SettingsMgr, ConfigMgr, EnvironmentsMgr`
- **Endpoints**:
  - `GET /api/v1/admin/variables` - List/get system variables
  - `PUT /api/v1/admin/variables` - Set system variable
  - `GET /api/v1/admin/configs` - Get system configurations
  - `GET /api/v1/admin/environments` - Get environment variables
  - `GET /api/v1/admin/version` - Get RAGFlow version

### 1.2 User Management Features (Regular API - `api/apps/user_app.py`)

These are regular user-facing features (not admin-only).

#### **A. User Authentication**
- **Endpoints**:
  - `POST /user/login` - User login
  - `GET /user/logout` - User logout
  - `GET /user/info` - Get current user profile
  - `POST /user/register` - User registration
  - `GET /user/login/channels` - Get available OAuth channels
  - `GET /user/login/<channel>` - OAuth login redirect
  - `GET /user/oauth/callback/<channel>` - OAuth callback
  - `GET /user/github_callback` - GitHub OAuth (deprecated)
  - `GET /user/feishu_callback` - Feishu OAuth

**Features**:
- Email/password authentication
- OAuth/OIDC support (GitHub, Feishu, configurable)
- JWT token-based sessions
- Access token management

#### **B. Password Management**
- **Endpoints**:
  - `GET /user/forget/captcha` - Get password reset captcha
  - `POST /user/forget/otp` - Send password reset OTP
  - `POST /user/forget/verify-otp` - Verify OTP
  - `POST /user/forget/reset-password` - Reset password

**Flow**:
1. User requests captcha
2. User solves captcha and requests OTP
3. OTP sent via email
4. User verifies OTP
5. User resets password

#### **C. User Profile Management**
- **Endpoints**:
  - `POST /user/setting` - Update user settings
  - `GET /user/info` - Get user profile
  - `GET /user/tenant_info` - Get tenant information
  - `POST /user/set_tenant_info` - Update tenant information

**Settings Include**:
- Nickname
- Avatar
- Language
- Color scheme
- Timezone
- Password change

#### **D. Tenant Management**
- **Location**: `api/apps/tenant_app.py`
- **Endpoints**:
  - `GET /tenant/list` - List user's tenants
  - `GET /tenant/<tenant_id>/user/list` - List tenant members
  - `POST /tenant/<tenant_id>/user` - Invite user to tenant
  - `DELETE /tenant/<tenant_id>/user/<user_id>` - Remove user from tenant
  - `PUT /tenant/agree/<tenant_id>` - Accept tenant invitation

**Tenant Roles**:
- `owner` - Tenant owner
- `admin` - Tenant admin
- `normal` - Regular member
- `invite` - Pending invitation

### 1.3 System Features (Regular API - `api/apps/system_app.py`)

These are system-level features accessible to regular users.

#### **A. System Information**
- **Endpoints**:
  - `GET /system/version` - Get application version
  - `GET /system/status` - Get system health status
  - `GET /system/healthz` - Health check endpoint
  - `GET /system/ping` - Ping endpoint
  - `GET /system/config` - Get system configuration

**Status Checks**:
- Document engine (Elasticsearch/Infinity)
- Storage (MinIO/S3)
- Database (MySQL/PostgreSQL)
- Redis
- Task executor heartbeats

#### **B. API Token Management**
- **Endpoints**:
  - `POST /system/new_token` - Generate new API token
  - `GET /system/token_list` - List API tokens
  - `DELETE /system/token/<token>` - Delete API token

---

## 2. Database Models & Schema

### 2.1 Core Tables

#### **User Table** (`user`)
```sql
- id (PK, VARCHAR(32))
- access_token (VARCHAR(255), indexed)
- nickname (VARCHAR(100), indexed)
- password (VARCHAR(255), indexed)
- email (VARCHAR(255), indexed, unique)
- avatar (TEXT)
- language (VARCHAR(32))
- color_schema (VARCHAR(32))
- timezone (VARCHAR(64))
- last_login_time (DATETIME, indexed)
- is_authenticated (CHAR(1), default '1')
- is_active (CHAR(1), default '1') -- Admin can toggle this
- is_anonymous (CHAR(1), default '0')
- login_channel (VARCHAR) -- 'password', 'github', 'feishu', etc.
- status (CHAR(1), default '1') -- 0=deleted, 1=active
- is_superuser (BOOLEAN, default false) -- Admin flag
- create_time, create_date, update_time, update_date
```

#### **Tenant Table** (`tenant`)
```sql
- id (PK, VARCHAR(32)) -- Same as owner user_id
- name (VARCHAR)
- llm_id, embd_id, rerank_id, asr_id, img2txt_id, tts_id
- parser_ids (JSON)
- status (CHAR(1))
- create_time, create_date, update_time, update_date
```

#### **UserTenant Table** (`user_tenant`)
```sql
- id (PK, VARCHAR(32))
- user_id (FK → user.id, indexed)
- tenant_id (FK → tenant.id, indexed)
- role (VARCHAR(32)) -- 'owner', 'admin', 'normal', 'invite'
- invited_by (VARCHAR(32), indexed)
- status (CHAR(1), default '1')
- create_time, create_date, update_time, update_date
```

#### **APIToken Table** (`api_token`)
```sql
- id (PK)
- tenant_id (FK → tenant.id)
- token (VARCHAR, unique)
- beta (VARCHAR(32))
- create_time, create_date, update_time, update_date
```

### 2.2 Database Access Patterns

**UserService** (`api/db/services/user_service.py`):
- `query()` - Query users with filters
- `query_user()` - Authenticate user
- `save()` - Create user (hashes password)
- `update_user()` - Update user
- `update_user_password()` - Change password
- `delete_user()` - Soft delete (status=0)
- `is_admin()` - Check if user is admin
- `get_all_users()` - Get all users (admin)

**TenantService** (`api/db/services/user_service.py`):
- `get_info_by()` - Get tenant info for user
- `get_joined_tenants_by_user_id()` - Get tenants user joined

**UserTenantService** (`api/db/services/user_service.py`):
- `get_by_tenant_id()` - Get tenant members
- `get_tenants_by_user_id()` - Get user's tenants
- `save()` - Create user-tenant relationship

---

## 3. Dependencies & Integration Points

### 3.1 How Other Features Depend on User/Admin Features

#### **A. Authentication Dependency**

**ALL features depend on authentication**:
- Every API endpoint uses `@login_required` decorator
- Authentication middleware in `api/apps/__init__.py::_load_user()`
- Validates JWT token or API token
- Sets `current_user` context

**Dependent Features**:
```
User Management (Auth)
    │
    ├─→ Knowledge Base Management
    ├─→ Document Management
    ├─→ Chat & Conversation
    ├─→ Agent System
    ├─→ File Management
    ├─→ Search
    ├─→ LLM Configuration
    └─→ All other features
```

**Code Pattern**:
```python
@manager.route("/some-endpoint")
@login_required  # Requires authentication
async def some_handler():
    user = current_user  # Available after auth
    # Use user.id, user.tenant_id, etc.
```

#### **B. User Context Dependency**

**Features that use user context**:
1. **Knowledge Base** - Filtered by `tenant_id`
2. **Documents** - Associated with user/tenant
3. **Chat** - Linked to user
4. **Agents** - Owned by user
5. **Files** - Stored per tenant
6. **API Tokens** - Generated per tenant

**Example**:
```python
# Knowledge base creation
kb = KnowledgebaseService.create(
    tenant_id=current_user.id,  # Uses authenticated user
    name=name
)
```

#### **C. Tenant Isolation**

**Multi-tenancy enforced**:
- All data queries filtered by `tenant_id`
- Users can belong to multiple tenants
- Tenant roles control access

**Dependent Features**:
- Knowledge bases are tenant-scoped
- Documents belong to knowledge bases (tenant-scoped)
- Chats are user-scoped but can access tenant KBs
- Agents are user-scoped

#### **D. Admin Features Dependencies**

**Admin features depend on**:
- User table (to check `is_superuser`)
- All other tables (to view/manage user data)

**Admin can access**:
- All user data
- All knowledge bases
- All agents
- All documents
- System services status

### 3.2 External Dependencies

#### **A. Redis**
- Session storage (Flask sessions)
- OTP storage (password reset)
- Captcha storage
- Task executor heartbeats
- Rate limiting

#### **B. Database (MySQL/PostgreSQL)**
- All user/tenant data
- All relationships

#### **C. Email Service**
- Password reset OTP
- Tenant invitations
- Admin notifications

---

## 4. Migration Strategy for .NET

### 4.1 Phase 1: Core User Management (.NET)

**Priority**: HIGH (Foundation for everything)

#### **Week 1-2: Database Layer**
- [ ] Set up Entity Framework Core
- [ ] Create User, Tenant, UserTenant, APIToken models
- [ ] Create DbContext
- [ ] Database migrations
- [ ] Connection pooling configuration

**Files to Reference**:
- `api/db/db_models.py` (User, Tenant, UserTenant models)
- `api/db/services/user_service.py` (Service methods)

#### **Week 3-4: Authentication Service**
- [ ] JWT token generation/validation
- [ ] Password hashing (BCrypt/Argon2)
- [ ] Authentication middleware
- [ ] Session management
- [ ] OAuth/OIDC integration

**Files to Reference**:
- `api/apps/__init__.py::_load_user()`
- `api/apps/user_app.py::login()`
- `api/utils/crypt.py`

#### **Week 5-6: User CRUD Operations**
- [ ] User registration
- [ ] User login/logout
- [ ] User profile management
- [ ] Password change
- [ ] User query/filter

**Files to Reference**:
- `api/apps/user_app.py`
- `api/db/services/user_service.py`

### 4.2 Phase 2: Password Reset & OAuth (.NET)

#### **Week 7-8: Password Reset Flow**
- [ ] Captcha generation
- [ ] OTP generation and email sending
- [ ] OTP verification
- [ ] Password reset endpoint

**Files to Reference**:
- `api/apps/user_app.py::forget_*` endpoints
- `api/utils/web_utils.py` (OTP, captcha)

#### **Week 9-10: OAuth Integration**
- [ ] OAuth channel configuration
- [ ] OAuth redirect handling
- [ ] OAuth callback processing
- [ ] User creation from OAuth

**Files to Reference**:
- `api/apps/user_app.py::oauth_*` endpoints
- `api/apps/auth/`

### 4.3 Phase 3: Tenant Management (.NET)

#### **Week 11-12: Tenant Operations**
- [ ] Tenant CRUD
- [ ] User-tenant relationship management
- [ ] Tenant invitation system
- [ ] Tenant role management

**Files to Reference**:
- `api/apps/tenant_app.py`
- `api/db/services/user_service.py::TenantService, UserTenantService`

### 4.4 Phase 4: Admin Features (.NET)

#### **Week 13-14: Admin Authentication**
- [ ] Admin login (separate from user login)
- [ ] Admin authorization middleware
- [ ] Admin session management

**Files to Reference**:
- `admin/server/auth.py`
- `admin/server/routes.py::login()`

#### **Week 15-16: Admin User Management**
- [ ] List all users
- [ ] Create user (admin)
- [ ] Delete user
- [ ] Activate/deactivate user
- [ ] Grant/revoke admin
- [ ] View user resources

**Files to Reference**:
- `admin/server/services.py::UserMgr`
- `admin/server/routes.py::*users*`

#### **Week 17-18: Admin Service Management**
- [ ] Service discovery
- [ ] Service health monitoring
- [ ] Service restart/shutdown

**Files to Reference**:
- `admin/server/services.py::ServiceMgr`
- `api/apps/system_app.py::status()`

#### **Week 19-20: Admin RBAC (If Enterprise)**
- [ ] Role CRUD
- [ ] Permission management
- [ ] User role assignment

**Files to Reference**:
- `admin/server/roles.py`

### 4.5 Phase 5: System Features (.NET)

#### **Week 21-22: System Endpoints**
- [ ] System status/health
- [ ] Version information
- [ ] Configuration endpoints
- [ ] API token management

**Files to Reference**:
- `api/apps/system_app.py`

---

## 5. .NET Implementation Details

### 5.1 Technology Stack Recommendations

**Framework**: ASP.NET Core 8.0
- Web API
- Built-in dependency injection
- Middleware pipeline
- JWT authentication

**ORM**: Entity Framework Core
- Code-first migrations
- LINQ queries
- Connection pooling

**Authentication**: 
- `Microsoft.AspNetCore.Authentication.JwtBearer` for JWT
- `BCrypt.Net-Next` for password hashing
- `Microsoft.AspNetCore.Authentication.OpenIdConnect` for OAuth

**Other Libraries**:
- `StackExchange.Redis` for Redis
- `MailKit` for email
- `SkiaSharp` or `ImageSharp` for captcha generation

### 5.2 Project Structure

```
RAGFlow.Admin/
├── Controllers/
│   ├── AdminController.cs
│   ├── UserController.cs
│   ├── TenantController.cs
│   └── SystemController.cs
├── Services/
│   ├── UserService.cs
│   ├── TenantService.cs
│   ├── AdminService.cs
│   ├── AuthService.cs
│   └── EmailService.cs
├── Models/
│   ├── User.cs
│   ├── Tenant.cs
│   ├── UserTenant.cs
│   └── DTOs/
├── Data/
│   ├── RAGFlowDbContext.cs
│   └── Repositories/
├── Middleware/
│   ├── AuthenticationMiddleware.cs
│   └── AdminAuthMiddleware.cs
└── Infrastructure/
    ├── JwtTokenService.cs
    ├── PasswordHasher.cs
    └── RedisService.cs
```

### 5.3 Key Implementation Patterns

#### **Authentication Middleware**
```csharp
public class AuthenticationMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var token = context.Request.Headers["Authorization"].FirstOrDefault();
        if (token != null)
        {
            // Validate JWT or API token
            // Set user context
        }
        await next(context);
    }
}
```

#### **Service Pattern**
```csharp
public class UserService
{
    private readonly RAGFlowDbContext _context;
    
    public async Task<User> AuthenticateAsync(string email, string password)
    {
        var user = await _context.Users
            .FirstOrDefaultAsync(u => u.Email == email && u.Status == "1");
        
        if (user != null && BCrypt.Verify(password, user.Password))
            return user;
        
        return null;
    }
}
```

### 5.4 Database Migration Strategy

**Option 1: Shared Database**
- .NET service accesses same MySQL/PostgreSQL
- Use Entity Framework migrations
- Coordinate with Python team on schema changes

**Option 2: Separate Tables**
- Create new tables with `_net` suffix
- Gradually migrate data
- More complex but safer

**Recommended**: Option 1 (Shared Database) for faster migration

---

## 6. Integration with Existing Python Code

### 6.1 During Migration

**Parallel Operation**:
- .NET service runs alongside Python
- API Gateway routes admin/user endpoints to .NET
- Other endpoints still use Python
- Shared database ensures data consistency

**API Gateway Configuration**:
```
/admin/* → .NET Service (Port 9381)
/user/* → .NET Service (Port 9381)
/tenant/* → .NET Service (Port 9381)
/system/* → .NET Service (Port 9381)
/* → Python Service (Port 9380)
```

### 6.2 Authentication Compatibility

**Shared JWT Secret**:
- Both Python and .NET use same secret
- Tokens generated by either service work with both
- Ensures seamless transition

**Shared Session Storage**:
- Use Redis for session storage
- Both services can read/write sessions

### 6.3 Data Consistency

**Transaction Management**:
- Use database transactions for critical operations
- Coordinate with Python team on schema changes
- Version database migrations

---

## 7. Testing Strategy

### 7.1 Unit Tests
- Service layer tests
- Authentication tests
- Password hashing tests
- OTP generation/verification tests

### 7.2 Integration Tests
- Database operations
- Redis operations
- Email sending
- OAuth flows

### 7.3 End-to-End Tests
- Complete user registration flow
- Password reset flow
- Admin user management flow
- Tenant management flow

### 7.4 Compatibility Tests
- Verify tokens from .NET work with Python
- Verify Python tokens work with .NET
- Test shared database access

---

## 8. Risk Mitigation

### 8.1 Authentication Risks
- **Risk**: Token incompatibility
- **Mitigation**: Use same JWT secret, test thoroughly

### 8.2 Data Consistency Risks
- **Risk**: Concurrent writes from Python and .NET
- **Mitigation**: Use database transactions, coordinate operations

### 8.3 Migration Risks
- **Risk**: Breaking existing functionality
- **Mitigation**: Gradual rollout, feature flags, canary deployments

### 8.4 Performance Risks
- **Risk**: .NET service slower than Python
- **Mitigation**: Performance testing, optimization, caching

---

## 9. Success Criteria

- [ ] All admin endpoints migrated to .NET
- [ ] All user management endpoints migrated to .NET
- [ ] Authentication works with both Python and .NET
- [ ] No data loss during migration
- [ ] Performance equal or better than Python
- [ ] All tests passing
- [ ] Documentation complete

---

## 10. Key Files Reference

### Admin Server
- `admin/server/routes.py` - All admin endpoints
- `admin/server/services.py` - Business logic
- `admin/server/auth.py` - Admin authentication
- `admin/server/roles.py` - RBAC (if enterprise)

### User Management
- `api/apps/user_app.py` - User endpoints
- `api/apps/tenant_app.py` - Tenant endpoints
- `api/apps/system_app.py` - System endpoints
- `api/db/services/user_service.py` - User/Tenant services
- `api/db/db_models.py` - Database models

### Authentication
- `api/apps/__init__.py` - Auth middleware
- `api/apps/auth/` - OAuth handlers
- `api/utils/crypt.py` - Encryption utilities
- `api/utils/web_utils.py` - OTP, email utilities

---

## 11. Next Steps

1. **Review & Approve**: Get stakeholder approval
2. **Setup .NET Project**: Create solution structure
3. **Database Setup**: Configure Entity Framework
4. **Implement Phase 1**: Core user management
5. **Test & Iterate**: Continuous testing
6. **Deploy & Monitor**: Gradual rollout

---

## Conclusion

This document provides a complete roadmap for migrating admin and user management features to .NET. The migration should be done incrementally, starting with core authentication, then user management, then admin features. Careful attention must be paid to maintaining compatibility with the existing Python codebase during the transition period.
