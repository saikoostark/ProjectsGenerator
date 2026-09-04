## 1. Project Title

RBAC and Audit Compliance Dashboard

## 2. Difficulty

Junior+

### Rationale
This project focuses on enterprise security, authorization architecture, and compliance reporting. The developer must design a granular Role-Based Access Control (RBAC) and Permission Inheritance system, enforce authorization middleware across API endpoints, and maintain an immutable, tamper-evident audit trail logging every administrative and data-access action.

## 3. Project Overview

The RBAC and Audit Compliance Dashboard is a full-stack enterprise administration tool. It enables security officers and administrators to define roles, assign fine-grained permissions (e.g., `users:read`, `billing:write`, `reports:export`), manage user-to-role mappings, and monitor system activity through an immutable audit log. Every sensitive API action triggers an auditable event with user metadata, IP address, timestamp, and success/failure status, presented in a searchable compliance dashboard.

## 4. Problem Statement

Enterprise applications require rigorous security controls and compliance auditing (e.g., SOC2, GDPR, HIPAA). 
- Simple boolean flags (`isAdmin: true/false`) are inadequate for complex enterprise hierarchies where different teams require distinct, overlapping access rights.
- Without a centralized authorization model, access checks become scattered across application code, leading to security bypass vulnerabilities.
- Without immutable audit logs, organizations cannot investigate security breaches or prove regulatory compliance during audits.

## 5. Proposed Solution

The proposed software consists of:
1. **Granular RBAC Engine**: A database schema and middleware that evaluates user permissions dynamically based on assigned roles.
2. **Authorization Middleware**: Intercepts HTTP requests, checks if the authenticated user possesses the required permission string, and rejects unauthorized requests with `403 Forbidden`.
3. **Immutable Audit Logger**: Intercepts sensitive write operations and security events, appending them to a dedicated audit trail table with cryptographic checksum validation to ensure log integrity.
4. **Admin Compliance Dashboard**: A React-based web dashboard to manage users, assign roles, inspect permissions, and filter audit logs by date, user, action, or severity.

## 6. Project Goal

To build a full-stack security dashboard featuring granular role-based access control, secure permission-checking middleware, and an immutable audit logging system for compliance tracking.

## 7. Core Workflow

```text
User Request (API Action)              Auth Middleware                  Audit Logger & DB
     │                                       │                                 │
     ├──1. HTTP Request (with JWT Token)────>│                                 │
     │                                       │──2. Decode & Verify Token       │
     │                                       │──3. Query User Roles/Perms      │
     │                                       │                                 │
     │                                       ├─────── 4. Has Permission? ──────┤
     │                                       │                                 │
     │                                       │[No]                             │ [Yes]
     │                                       ▼                                 ▼
     │<─────5. Return 403 Forbidden──────────┤                      6. Execute API Controller
     │                                       │                      7. Append Immutable Audit Log
     │                                       │<────────────────────────────────┤
     │<─────8. Return Success Response───────┤
```

## 8. Functional Requirements

### RBAC & Permission Management
- **Roles & Permissions**: CRUD operations for roles (e.g., `Admin`, `Manager`, `Auditor`) and atomic permissions (e.g., `document:create`, `document:delete`).
- **User-Role Assignment**: Assign and revoke multiple roles per user.
- **Permission Middleware**: Enforce permission checks on backend API routes.

### Audit Logging & Compliance
- **Event Logging**: Automatically log security-relevant events (Login, Password Reset, Role Change, Data Deletion, Permission Denied).
- **Immutable Trail**: Ensure audit logs cannot be edited or deleted through the application interface.
- **Searchable Dashboard**: Filter audit logs by date range, user ID, event type, and IP address.

## 9. Non-Functional Requirements

### Security & Integrity
- **Tamper-Evident Logs**: Each audit log entry can include a cryptographic hash linking to the previous log entry (a hash chain) to detect database tampering.

## 10. Main Entities / Data Model

### User
- **ID**: UUID.
- **Email**: String.
- **PasswordHash**: String.

### Role & Permission
- **Role**: ID, Name, Description.
- **Permission**: ID, Name (e.g., `users:write`).
- **RolePermission**: Mapping table linking Roles to Permissions.
- **UserRole**: Mapping table linking Users to Roles.

### AuditLog
- **ID**: UUID.
- **Timestamp**: DateTime.
- **ActorID**: UUID (User who performed the action).
- **Action**: String (e.g., `ROLE_ASSIGNED`).
- **TargetResource**: String.
- **IpAddress**: String.
- **Status**: Enum (SUCCESS, FAILURE).
- **Checksum**: String (SHA-256 hash chaining).

## 11. System Components

- **Frontend Dashboard**: React/Vue admin panel for managing users, roles, and viewing audit logs.
- **API Server**: Backend handling authentication, RBAC authorization, and business logic.
- **Audit Logger Service**: Middleware writing tamper-evident event logs to the database.
- **Database**: PostgreSQL or SQLite storing users, RBAC mappings, and audit records.

## 12. Important Technical Challenges

### Designing Flexible RBAC Schemas
- **Challenge**: Users have roles, and roles have permissions. Evaluating whether a user has a specific permission requires traversing user-role-permission join tables efficiently.
- **Concepts**: Database join optimization, permission caching.

### Tamper-Evident Audit Chains
- **Challenge**: Ensuring that log records have not been altered in the database requires building a cryptographic hash chain similar to a blockchain.
- **Concepts**: Cryptographic hashing (SHA-256), previous-hash chaining.

## 13. Suggested Technology Areas

- **Backend**: Node.js (Express) or Python (FastAPI).
- **Frontend**: React, Tailwind CSS, Lucide icons.
- **Database**: PostgreSQL or SQLite.

## 14. Skills and Knowledge Gained

### Security & Architecture
- Designing robust Role-Based Access Control (RBAC) systems.
- Implementing authorization middleware and permission verification.
- Building compliance-ready, tamper-evident audit logging mechanisms.

## 15. Recommended Development Phases

1. **Phase 1 - RBAC Database Schema**: Design user, role, and permission tables and write seed data.
2. **Phase 2 - Authorization Middleware**: Implement JWT authentication and permission-checking middleware for backend routes.
3. **Phase 3 - Audit Logging Service**: Build the audit logger that records events and computes cryptographic hash chains.
4. **Phase 4 - Admin Dashboard**: Build the React interface for managing users, assigning roles, and viewing searchable audit logs.

## 16. Testing Requirements

* **Unit Tests**: Test permission evaluation logic (e.g., verifying users inherit correct permissions from multiple roles).
* **Security Tests**: Verify that users without required permissions receive `403 Forbidden` responses when accessing protected endpoints.

## 17. Security Considerations

* **Principle of Least Privilege**: Ensure default user accounts are created with zero administrative privileges.
* **Log Integrity**: Protect audit log tables from update and delete operations at the application level.

## 18. Possible Extensions

* **Attribute-Based Access Control (ABAC)**: Extend RBAC to support contextual attributes (e.g., "Users can edit documents only during working hours and from office IPs").
* **Export Compliance Reports**: Generate downloadable PDF or CSV compliance reports for auditors.

## 19. Learning Questions

* What is the difference between Authentication and Authorization?
* Why is RBAC superior to hardcoding boolean permission flags in user records?
* How does cryptographic hash chaining ensure the integrity of audit logs?

## 20. Completion Criteria

* [ ] Users can be assigned roles and permissions via the admin interface.
* [ ] Backend API routes correctly enforce permission middleware, returning `403` when unauthorized.
* [ ] Sensitive actions automatically generate audit log entries.
* [ ] Audit logs include timestamps, actor IDs, IP addresses, and cryptographic check hashes.
* [ ] The audit log dashboard supports filtering and searching by user, action, and date.