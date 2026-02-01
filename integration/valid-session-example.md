# Valid Session Example

This document provides a complete, realistic example of a valid agent session from start to finish, showing actual file contents at each step.

## Scenario

**Task**: Add a user authentication feature to an existing project
**Session ID**: `session-auth-2026-02-01-001`
**Workspace**: `/workspace/my-project`

---

## Step 1: Session Initialization

### 1.1 Verify AGENT_LINK.md Exists

**File**: `/workspace/my-project/AGENT_LINK.md`

```markdown
# Agent Link

This workspace is connected to the AI Agent system.

## Configuration

- **Boot Repository**: ../agent-boot
- **Contract Version**: 1.0.0
- **Created**: 2026-01-15T10:00:00Z

## Allowed Operations

- Read all files
- Write to src/, tests/, docs/
- Create branches with prefix: feature/, fix/, agent/
- Commit to feature branches
- Create pull requests

## Restrictions

- Cannot modify .github/workflows/
- Cannot delete configuration files
```

**Validation**: File exists ✓

### 1.2 Load and Validate STATE.json

**File**: `/workspace/my-project/STATE.json` (before session)

```json
{
  "sessionId": "session-prev-2026-01-31-003",
  "status": "completed",
  "lastUpdated": "2026-01-31T17:45:00Z",
  "currentTask": null,
  "checkpoint": null,
  "errors": []
}
```

**Validation**: Schema valid ✓

### 1.3 Initialize New Session

**File**: `/workspace/my-project/STATE.json` (after initialization)

```json
{
  "sessionId": "session-auth-2026-02-01-001",
  "status": "running",
  "lastUpdated": "2026-02-01T09:00:00Z",
  "currentTask": "Add user authentication feature",
  "checkpoint": {
    "phase": "initialization",
    "completedSteps": []
  },
  "errors": []
}
```

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:00:00.000Z",
  "level": "INFO",
  "event": "SESSION_START",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "start",
  "success": true,
  "details": {
    "contractVersion": "1.0.0",
    "bootVersion": "1.0.2",
    "workspace": "/workspace/my-project",
    "task": "Add user authentication feature"
  }
}
```

---

## Step 2: Read Task from TODO.json

### 2.1 Read TODO.json

**File**: `/workspace/my-project/TODO.json`

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-01-31T15:00:00Z",
  "tasks": [
    {
      "id": "task-001",
      "title": "Add user authentication feature",
      "description": "Implement login/logout functionality with JWT tokens",
      "status": "pending",
      "priority": "high",
      "assignedSession": null,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-01-30T10:00:00Z"
    },
    {
      "id": "task-002",
      "title": "Write unit tests for auth module",
      "description": "Cover login, logout, and token refresh flows",
      "status": "pending",
      "priority": "medium",
      "assignedSession": null,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-01-30T10:00:00Z"
    }
  ]
}
```

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:00:05.000Z",
  "level": "INFO",
  "event": "FILE_READ",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "read",
  "path": "TODO.json",
  "success": true,
  "duration_ms": 5
}
```

### 2.2 Claim Task

**File**: `/workspace/my-project/TODO.json` (after claiming)

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-02-01T09:00:10Z",
  "tasks": [
    {
      "id": "task-001",
      "title": "Add user authentication feature",
      "description": "Implement login/logout functionality with JWT tokens",
      "status": "in_progress",
      "priority": "high",
      "assignedSession": "session-auth-2026-02-01-001",
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-02-01T09:00:10Z"
    },
    {
      "id": "task-002",
      "title": "Write unit tests for auth module",
      "description": "Cover login, logout, and token refresh flows",
      "status": "pending",
      "priority": "medium",
      "assignedSession": null,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-01-30T10:00:00Z"
    }
  ]
}
```

---

## Step 3: Create Feature Branch

### 3.1 Validate Branch Name

**Proposed Branch**: `feature/add-user-auth`

**Validation**:
- Matches pattern `^(feature|fix|agent|update)/[a-z0-9-]+$` ✓
- Length < 100 characters ✓
- Not a protected branch name ✓

### 3.2 Create Branch

```bash
git checkout -b feature/add-user-auth
```

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:00:15.000Z",
  "level": "INFO",
  "event": "GIT_BRANCH",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "create_branch",
  "repository": "/workspace/my-project",
  "success": true,
  "details": {
    "branch": "feature/add-user-auth",
    "baseBranch": "main"
  }
}
```

### 3.3 Update STATE.json

**File**: `/workspace/my-project/STATE.json`

```json
{
  "sessionId": "session-auth-2026-02-01-001",
  "status": "running",
  "lastUpdated": "2026-02-01T09:00:15Z",
  "currentTask": "Add user authentication feature",
  "checkpoint": {
    "phase": "development",
    "completedSteps": ["initialization", "task_claimed", "branch_created"],
    "branch": "feature/add-user-auth"
  },
  "errors": []
}
```

---

## Step 4: Implement Authentication Module

### 4.1 Read Existing Project Structure

**Files Read**:
- `src/index.ts`
- `src/config.ts`
- `package.json`

### 4.2 Create Auth Module

**File**: `/workspace/my-project/src/auth/index.ts`

```typescript
import jwt from 'jsonwebtoken';
import { config } from '../config';

export interface User {
  id: string;
  email: string;
  name: string;
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

export async function login(email: string, password: string): Promise<AuthTokens> {
  // Validate credentials (implementation would check database)
  const user = await validateCredentials(email, password);

  if (!user) {
    throw new AuthError('Invalid credentials');
  }

  return generateTokens(user);
}

export async function logout(refreshToken: string): Promise<void> {
  // Invalidate refresh token
  await invalidateToken(refreshToken);
}

export function verifyAccessToken(token: string): User {
  try {
    const payload = jwt.verify(token, config.jwtSecret) as User;
    return payload;
  } catch (error) {
    throw new AuthError('Invalid or expired token');
  }
}

export async function refreshAccessToken(refreshToken: string): Promise<AuthTokens> {
  const user = await validateRefreshToken(refreshToken);

  if (!user) {
    throw new AuthError('Invalid refresh token');
  }

  return generateTokens(user);
}

function generateTokens(user: User): AuthTokens {
  const accessToken = jwt.sign(
    { id: user.id, email: user.email, name: user.name },
    config.jwtSecret,
    { expiresIn: '15m' }
  );

  const refreshToken = jwt.sign(
    { id: user.id },
    config.jwtRefreshSecret,
    { expiresIn: '7d' }
  );

  return { accessToken, refreshToken };
}

async function validateCredentials(email: string, password: string): Promise<User | null> {
  // Database lookup implementation
  return null; // Placeholder
}

async function validateRefreshToken(token: string): Promise<User | null> {
  // Validate and lookup user from refresh token
  return null; // Placeholder
}

async function invalidateToken(token: string): Promise<void> {
  // Add token to blacklist
}

export class AuthError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'AuthError';
  }
}
```

**Validations Passed**:
- V-001: Path allowlist ✓ (src/ is allowed)
- V-002: File size 1.8 KB < 1 MB ✓
- V-004: No secrets detected ✓

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:05:00.000Z",
  "level": "INFO",
  "event": "FILE_WRITE",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "write",
  "path": "src/auth/index.ts",
  "success": true,
  "duration_ms": 45,
  "details": {
    "size_bytes": 1843,
    "validations_passed": ["V-001", "V-002", "V-004"]
  }
}
```

### 4.3 Create Auth Types

**File**: `/workspace/my-project/src/auth/types.ts`

```typescript
export interface LoginRequest {
  email: string;
  password: string;
}

export interface LoginResponse {
  user: {
    id: string;
    email: string;
    name: string;
  };
  tokens: {
    accessToken: string;
    refreshToken: string;
  };
}

export interface RefreshRequest {
  refreshToken: string;
}

export interface RefreshResponse {
  accessToken: string;
  refreshToken: string;
}

export interface AuthenticatedRequest {
  user: {
    id: string;
    email: string;
    name: string;
  };
}
```

**Validations Passed**: V-001 ✓, V-002 ✓, V-004 ✓

### 4.4 Update STATE.json

**File**: `/workspace/my-project/STATE.json`

```json
{
  "sessionId": "session-auth-2026-02-01-001",
  "status": "running",
  "lastUpdated": "2026-02-01T09:10:00Z",
  "currentTask": "Add user authentication feature",
  "checkpoint": {
    "phase": "development",
    "completedSteps": [
      "initialization",
      "task_claimed",
      "branch_created",
      "auth_module_created",
      "auth_types_created"
    ],
    "branch": "feature/add-user-auth",
    "filesCreated": [
      "src/auth/index.ts",
      "src/auth/types.ts"
    ]
  },
  "errors": []
}
```

---

## Step 5: Update CHANGELOG.md

**File**: `/workspace/my-project/CHANGELOG.md` (before)

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## [1.0.0] - 2026-01-15

### Added
- Initial project setup
- Basic API structure
```

**File**: `/workspace/my-project/CHANGELOG.md` (after)

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- User authentication module with JWT tokens
- Login, logout, and token refresh functionality
- AuthError class for authentication errors

## [1.0.0] - 2026-01-15

### Added
- Initial project setup
- Basic API structure
```

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:15:00.000Z",
  "level": "INFO",
  "event": "FILE_WRITE",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "write",
  "path": "CHANGELOG.md",
  "success": true,
  "details": {
    "size_bytes": 456,
    "validations_passed": ["V-001", "V-002"]
  }
}
```

---

## Step 6: Commit Changes

### 6.1 Stage Files

```bash
git add src/auth/index.ts src/auth/types.ts CHANGELOG.md STATE.json TODO.json
```

### 6.2 Pre-Commit Validations

**Validations Run**:
- V-006: STATE.json updated ✓
- V-007: No secrets in staged files ✓
- V-008: Staged files size (total 4.2 KB) < 50 MB ✓

### 6.3 Create Commit

```bash
git commit -m "Add user authentication module with JWT tokens

- Implement login/logout functions
- Add token generation and validation
- Create type definitions for auth requests/responses
- Update CHANGELOG.md"
```

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:20:00.000Z",
  "level": "INFO",
  "event": "GIT_COMMIT",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "commit",
  "repository": "/workspace/my-project",
  "success": true,
  "details": {
    "commitHash": "a1b2c3d4e5f6g7h8i9j0",
    "branch": "feature/add-user-auth",
    "message": "Add user authentication module with JWT tokens",
    "filesChanged": 5,
    "insertions": 125,
    "deletions": 0
  }
}
```

---

## Step 7: Push and Create Pull Request

### 7.1 Push Branch

```bash
git push -u origin feature/add-user-auth
```

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:21:00.000Z",
  "level": "INFO",
  "event": "GIT_PUSH",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "push",
  "repository": "/workspace/my-project",
  "success": true,
  "details": {
    "branch": "feature/add-user-auth",
    "remote": "origin",
    "commits": 1
  }
}
```

### 7.2 Validate PR Requirements

- V-009: Branch name follows convention ✓
- V-010: Description not empty ✓
- V-011: Title not empty ✓

### 7.3 Create Pull Request

**PR Title**: "Add user authentication module"

**PR Description**:
```markdown
## Summary

This PR adds a user authentication module with JWT token support.

### Changes
- New `src/auth/index.ts` with login, logout, and token refresh functions
- New `src/auth/types.ts` with TypeScript interfaces
- Updated CHANGELOG.md

### Testing
- Manual testing of token generation
- Types compile successfully

## Related
- Closes task-001
```

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:22:00.000Z",
  "level": "INFO",
  "event": "PR_CREATE",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "create_pr",
  "repository": "/workspace/my-project",
  "success": true,
  "details": {
    "prNumber": 42,
    "title": "Add user authentication module",
    "sourceBranch": "feature/add-user-auth",
    "targetBranch": "main"
  }
}
```

---

## Step 8: Complete Session

### 8.1 Update TODO.json

**File**: `/workspace/my-project/TODO.json` (final)

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-02-01T09:25:00Z",
  "tasks": [
    {
      "id": "task-001",
      "title": "Add user authentication feature",
      "description": "Implement login/logout functionality with JWT tokens",
      "status": "completed",
      "priority": "high",
      "assignedSession": "session-auth-2026-02-01-001",
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-02-01T09:25:00Z",
      "completedAt": "2026-02-01T09:25:00Z",
      "prNumber": 42
    },
    {
      "id": "task-002",
      "title": "Write unit tests for auth module",
      "description": "Cover login, logout, and token refresh flows",
      "status": "pending",
      "priority": "medium",
      "assignedSession": null,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-01-30T10:00:00Z"
    }
  ]
}
```

### 8.2 Update STATE.json (Final)

**File**: `/workspace/my-project/STATE.json` (final)

```json
{
  "sessionId": "session-auth-2026-02-01-001",
  "status": "completed",
  "lastUpdated": "2026-02-01T09:25:00Z",
  "currentTask": null,
  "checkpoint": {
    "phase": "completed",
    "completedSteps": [
      "initialization",
      "task_claimed",
      "branch_created",
      "auth_module_created",
      "auth_types_created",
      "changelog_updated",
      "committed",
      "pushed",
      "pr_created",
      "task_completed"
    ],
    "branch": "feature/add-user-auth",
    "prNumber": 42
  },
  "errors": []
}
```

### 8.3 Session End

**Audit Log Entry**:
```json
{
  "timestamp": "2026-02-01T09:25:00.000Z",
  "level": "INFO",
  "event": "SESSION_END",
  "sessionId": "session-auth-2026-02-01-001",
  "operation": "complete",
  "success": true,
  "details": {
    "status": "completed",
    "duration_minutes": 25,
    "summary": {
      "filesRead": 5,
      "filesWritten": 5,
      "commits": 1,
      "prsCreated": 1,
      "tasksCompleted": 1,
      "validationsPassed": 18,
      "validationsFailed": 0,
      "errors": 0
    }
  }
}
```

---

## Session Summary

| Metric | Value |
|--------|-------|
| Session ID | session-auth-2026-02-01-001 |
| Duration | 25 minutes |
| Files Created | 2 |
| Files Modified | 3 |
| Commits | 1 |
| Pull Requests | 1 |
| Tasks Completed | 1 |
| Validations Passed | 18 |
| Errors | 0 |

This session demonstrates a complete, valid workflow following all contract requirements.
