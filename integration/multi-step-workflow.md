# Multi-Step Workflow Example

This document demonstrates how a task spans multiple sessions, showing how STATE.json evolves and how work is resumed across sessions.

## Scenario

**Task**: Implement a complete user profile feature with database, API, and UI components

This task is too large for a single session and will be split across three sessions:
1. **Session 1**: Read task, plan, create branch, implement database layer
2. **Session 2**: Resume, implement API endpoints
3. **Session 3**: Resume, implement UI, complete task

---

## Session 1: Planning and Database Layer

### Session Start

**Session ID**: `session-profile-001`
**Start Time**: 2026-02-01T09:00:00Z

### 1.1 Initial STATE.json

**File**: `/workspace/my-project/STATE.json` (at start of Session 1)

```json
{
  "sessionId": "session-prev-000",
  "status": "completed",
  "lastUpdated": "2026-01-31T17:00:00Z",
  "currentTask": null,
  "checkpoint": null,
  "errors": []
}
```

### 1.2 Read and Claim Task

**File**: `/workspace/my-project/TODO.json` (initial)

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-01-31T15:00:00Z",
  "tasks": [
    {
      "id": "task-profile-001",
      "title": "Implement user profile feature",
      "description": "Add complete user profile with avatar, settings, and preferences. Includes database schema, API endpoints, and React UI components.",
      "status": "pending",
      "priority": "high",
      "assignedSession": null,
      "estimatedSessions": 3,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-01-30T10:00:00Z",
      "subtasks": [
        {"id": "sub-001", "title": "Database schema and migrations", "status": "pending"},
        {"id": "sub-002", "title": "API endpoints", "status": "pending"},
        {"id": "sub-003", "title": "React UI components", "status": "pending"}
      ]
    }
  ]
}
```

### 1.3 Update STATE.json with Plan

**File**: `/workspace/my-project/STATE.json` (after planning)

```json
{
  "sessionId": "session-profile-001",
  "status": "running",
  "lastUpdated": "2026-02-01T09:05:00Z",
  "currentTask": "task-profile-001",
  "checkpoint": {
    "phase": "planning",
    "taskId": "task-profile-001",
    "plan": {
      "session1": ["Create branch", "Database schema", "Migrations"],
      "session2": ["API endpoints", "Validation", "Tests"],
      "session3": ["React components", "Integration", "Final testing"]
    },
    "currentSubtask": "sub-001",
    "completedSubtasks": []
  },
  "errors": []
}
```

### 1.4 Create Feature Branch

```bash
git checkout -b feature/user-profile
```

### 1.5 Implement Database Schema

**File**: `/workspace/my-project/src/db/schema/profile.ts`

```typescript
import { pgTable, uuid, varchar, text, timestamp, jsonb } from 'drizzle-orm/pg-core';

export const userProfiles = pgTable('user_profiles', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().unique(),
  displayName: varchar('display_name', { length: 100 }),
  bio: text('bio'),
  avatarUrl: varchar('avatar_url', { length: 500 }),
  preferences: jsonb('preferences').default({}),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});

export type UserProfile = typeof userProfiles.$inferSelect;
export type NewUserProfile = typeof userProfiles.$inferInsert;
```

**File**: `/workspace/my-project/src/db/migrations/002_user_profiles.sql`

```sql
-- Migration: Create user_profiles table
-- Created: 2026-02-01

CREATE TABLE IF NOT EXISTS user_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    preferences JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_user_profiles_user_id ON user_profiles(user_id);

-- Trigger to auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_user_profiles_updated_at
    BEFORE UPDATE ON user_profiles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### 1.6 Commit Progress

```bash
git add src/db/schema/profile.ts src/db/migrations/002_user_profiles.sql
git commit -m "Add user profile database schema and migrations

- Create user_profiles table with avatar, bio, and preferences
- Add migration with indexes and auto-update trigger
- Define TypeScript types for profile model"
```

### 1.7 Update STATE.json Before Ending

**File**: `/workspace/my-project/STATE.json` (at end of Session 1)

```json
{
  "sessionId": "session-profile-001",
  "status": "paused",
  "lastUpdated": "2026-02-01T10:30:00Z",
  "currentTask": "task-profile-001",
  "checkpoint": {
    "phase": "development",
    "taskId": "task-profile-001",
    "plan": {
      "session1": ["Create branch", "Database schema", "Migrations"],
      "session2": ["API endpoints", "Validation", "Tests"],
      "session3": ["React components", "Integration", "Final testing"]
    },
    "currentSubtask": "sub-002",
    "completedSubtasks": ["sub-001"],
    "branch": "feature/user-profile",
    "commits": [
      {
        "hash": "abc1234",
        "message": "Add user profile database schema and migrations"
      }
    ],
    "filesCreated": [
      "src/db/schema/profile.ts",
      "src/db/migrations/002_user_profiles.sql"
    ],
    "nextSteps": [
      "Implement ProfileService class",
      "Create API routes for profile CRUD",
      "Add input validation"
    ]
  },
  "errors": []
}
```

### 1.8 Update TODO.json

**File**: `/workspace/my-project/TODO.json` (at end of Session 1)

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-02-01T10:30:00Z",
  "tasks": [
    {
      "id": "task-profile-001",
      "title": "Implement user profile feature",
      "description": "Add complete user profile with avatar, settings, and preferences. Includes database schema, API endpoints, and React UI components.",
      "status": "in_progress",
      "priority": "high",
      "assignedSession": "session-profile-001",
      "estimatedSessions": 3,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-02-01T10:30:00Z",
      "subtasks": [
        {"id": "sub-001", "title": "Database schema and migrations", "status": "completed"},
        {"id": "sub-002", "title": "API endpoints", "status": "pending"},
        {"id": "sub-003", "title": "React UI components", "status": "pending"}
      ]
    }
  ]
}
```

### Session 1 End

**Duration**: 1.5 hours
**Progress**: 33% (1 of 3 subtasks complete)

---

## Session 2: API Endpoints

### Session Start

**Session ID**: `session-profile-002`
**Start Time**: 2026-02-01T14:00:00Z

### 2.1 Resume from STATE.json

Agent reads STATE.json and resumes from checkpoint.

**File**: `/workspace/my-project/STATE.json` (at start of Session 2)

```json
{
  "sessionId": "session-profile-002",
  "status": "running",
  "lastUpdated": "2026-02-01T14:00:00Z",
  "currentTask": "task-profile-001",
  "checkpoint": {
    "phase": "development",
    "taskId": "task-profile-001",
    "plan": {
      "session1": ["Create branch", "Database schema", "Migrations"],
      "session2": ["API endpoints", "Validation", "Tests"],
      "session3": ["React components", "Integration", "Final testing"]
    },
    "currentSubtask": "sub-002",
    "completedSubtasks": ["sub-001"],
    "branch": "feature/user-profile",
    "commits": [
      {
        "hash": "abc1234",
        "message": "Add user profile database schema and migrations"
      }
    ],
    "filesCreated": [
      "src/db/schema/profile.ts",
      "src/db/migrations/002_user_profiles.sql"
    ],
    "resumedFrom": "session-profile-001",
    "nextSteps": [
      "Implement ProfileService class",
      "Create API routes for profile CRUD",
      "Add input validation"
    ]
  },
  "errors": []
}
```

### 2.2 Checkout Existing Branch

```bash
git checkout feature/user-profile
git pull origin feature/user-profile
```

### 2.3 Implement Profile Service

**File**: `/workspace/my-project/src/services/profile.service.ts`

```typescript
import { db } from '../db';
import { userProfiles, UserProfile, NewUserProfile } from '../db/schema/profile';
import { eq } from 'drizzle-orm';

export class ProfileService {
  async getProfile(userId: string): Promise<UserProfile | null> {
    const profiles = await db
      .select()
      .from(userProfiles)
      .where(eq(userProfiles.userId, userId))
      .limit(1);

    return profiles[0] || null;
  }

  async createProfile(data: NewUserProfile): Promise<UserProfile> {
    const [profile] = await db
      .insert(userProfiles)
      .values(data)
      .returning();

    return profile;
  }

  async updateProfile(
    userId: string,
    updates: Partial<Omit<UserProfile, 'id' | 'userId' | 'createdAt'>>
  ): Promise<UserProfile | null> {
    const [profile] = await db
      .update(userProfiles)
      .set({ ...updates, updatedAt: new Date() })
      .where(eq(userProfiles.userId, userId))
      .returning();

    return profile || null;
  }

  async deleteProfile(userId: string): Promise<boolean> {
    const result = await db
      .delete(userProfiles)
      .where(eq(userProfiles.userId, userId));

    return result.rowCount > 0;
  }

  async updateAvatar(userId: string, avatarUrl: string): Promise<UserProfile | null> {
    return this.updateProfile(userId, { avatarUrl });
  }

  async updatePreferences(
    userId: string,
    preferences: Record<string, unknown>
  ): Promise<UserProfile | null> {
    return this.updateProfile(userId, { preferences });
  }
}

export const profileService = new ProfileService();
```

### 2.4 Implement API Routes

**File**: `/workspace/my-project/src/routes/profile.routes.ts`

```typescript
import { Router } from 'express';
import { profileService } from '../services/profile.service';
import { authenticate } from '../middleware/auth';
import { validateBody } from '../middleware/validation';
import { profileSchema, updateProfileSchema, preferencesSchema } from '../schemas/profile';

const router = Router();

// All routes require authentication
router.use(authenticate);

// GET /api/profile - Get current user's profile
router.get('/', async (req, res) => {
  try {
    const profile = await profileService.getProfile(req.user.id);

    if (!profile) {
      return res.status(404).json({ error: 'Profile not found' });
    }

    res.json(profile);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch profile' });
  }
});

// POST /api/profile - Create profile for current user
router.post('/', validateBody(profileSchema), async (req, res) => {
  try {
    const profile = await profileService.createProfile({
      userId: req.user.id,
      ...req.body,
    });

    res.status(201).json(profile);
  } catch (error) {
    if (error.code === '23505') {
      return res.status(409).json({ error: 'Profile already exists' });
    }
    res.status(500).json({ error: 'Failed to create profile' });
  }
});

// PATCH /api/profile - Update current user's profile
router.patch('/', validateBody(updateProfileSchema), async (req, res) => {
  try {
    const profile = await profileService.updateProfile(req.user.id, req.body);

    if (!profile) {
      return res.status(404).json({ error: 'Profile not found' });
    }

    res.json(profile);
  } catch (error) {
    res.status(500).json({ error: 'Failed to update profile' });
  }
});

// PUT /api/profile/preferences - Update preferences
router.put('/preferences', validateBody(preferencesSchema), async (req, res) => {
  try {
    const profile = await profileService.updatePreferences(
      req.user.id,
      req.body.preferences
    );

    if (!profile) {
      return res.status(404).json({ error: 'Profile not found' });
    }

    res.json(profile);
  } catch (error) {
    res.status(500).json({ error: 'Failed to update preferences' });
  }
});

// DELETE /api/profile - Delete current user's profile
router.delete('/', async (req, res) => {
  try {
    const deleted = await profileService.deleteProfile(req.user.id);

    if (!deleted) {
      return res.status(404).json({ error: 'Profile not found' });
    }

    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: 'Failed to delete profile' });
  }
});

export default router;
```

### 2.5 Commit Progress

```bash
git add src/services/profile.service.ts src/routes/profile.routes.ts
git commit -m "Add profile service and API endpoints

- Implement ProfileService with CRUD operations
- Add REST API routes for profile management
- Include authentication middleware
- Add input validation"
```

### 2.6 Update STATE.json Before Ending

**File**: `/workspace/my-project/STATE.json` (at end of Session 2)

```json
{
  "sessionId": "session-profile-002",
  "status": "paused",
  "lastUpdated": "2026-02-01T16:00:00Z",
  "currentTask": "task-profile-001",
  "checkpoint": {
    "phase": "development",
    "taskId": "task-profile-001",
    "plan": {
      "session1": ["Create branch", "Database schema", "Migrations"],
      "session2": ["API endpoints", "Validation", "Tests"],
      "session3": ["React components", "Integration", "Final testing"]
    },
    "currentSubtask": "sub-003",
    "completedSubtasks": ["sub-001", "sub-002"],
    "branch": "feature/user-profile",
    "commits": [
      {
        "hash": "abc1234",
        "message": "Add user profile database schema and migrations"
      },
      {
        "hash": "def5678",
        "message": "Add profile service and API endpoints"
      }
    ],
    "filesCreated": [
      "src/db/schema/profile.ts",
      "src/db/migrations/002_user_profiles.sql",
      "src/services/profile.service.ts",
      "src/routes/profile.routes.ts"
    ],
    "resumedFrom": "session-profile-001",
    "nextSteps": [
      "Create ProfilePage React component",
      "Add ProfileForm for editing",
      "Implement avatar upload",
      "Create PR and complete task"
    ]
  },
  "errors": []
}
```

### 2.7 Update TODO.json

**File**: `/workspace/my-project/TODO.json` (at end of Session 2)

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-02-01T16:00:00Z",
  "tasks": [
    {
      "id": "task-profile-001",
      "title": "Implement user profile feature",
      "description": "Add complete user profile with avatar, settings, and preferences. Includes database schema, API endpoints, and React UI components.",
      "status": "in_progress",
      "priority": "high",
      "assignedSession": "session-profile-002",
      "estimatedSessions": 3,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-02-01T16:00:00Z",
      "subtasks": [
        {"id": "sub-001", "title": "Database schema and migrations", "status": "completed"},
        {"id": "sub-002", "title": "API endpoints", "status": "completed"},
        {"id": "sub-003", "title": "React UI components", "status": "pending"}
      ],
      "sessionHistory": ["session-profile-001", "session-profile-002"]
    }
  ]
}
```

### Session 2 End

**Duration**: 2 hours
**Progress**: 67% (2 of 3 subtasks complete)

---

## Session 3: UI Components and Completion

### Session Start

**Session ID**: `session-profile-003`
**Start Time**: 2026-02-02T09:00:00Z

### 3.1 Resume from STATE.json

**File**: `/workspace/my-project/STATE.json` (at start of Session 3)

```json
{
  "sessionId": "session-profile-003",
  "status": "running",
  "lastUpdated": "2026-02-02T09:00:00Z",
  "currentTask": "task-profile-001",
  "checkpoint": {
    "phase": "development",
    "taskId": "task-profile-001",
    "plan": {
      "session1": ["Create branch", "Database schema", "Migrations"],
      "session2": ["API endpoints", "Validation", "Tests"],
      "session3": ["React components", "Integration", "Final testing"]
    },
    "currentSubtask": "sub-003",
    "completedSubtasks": ["sub-001", "sub-002"],
    "branch": "feature/user-profile",
    "commits": [
      {"hash": "abc1234", "message": "Add user profile database schema and migrations"},
      {"hash": "def5678", "message": "Add profile service and API endpoints"}
    ],
    "filesCreated": [
      "src/db/schema/profile.ts",
      "src/db/migrations/002_user_profiles.sql",
      "src/services/profile.service.ts",
      "src/routes/profile.routes.ts"
    ],
    "resumedFrom": "session-profile-002"
  },
  "errors": []
}
```

### 3.2 Implement React Components

**File**: `/workspace/my-project/src/components/Profile/ProfilePage.tsx`

```typescript
import React from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { profileApi } from '../../api/profile';
import { ProfileForm } from './ProfileForm';
import { AvatarUpload } from './AvatarUpload';
import { PreferencesPanel } from './PreferencesPanel';

export const ProfilePage: React.FC = () => {
  const queryClient = useQueryClient();

  const { data: profile, isLoading, error } = useQuery({
    queryKey: ['profile'],
    queryFn: profileApi.getProfile,
  });

  const updateMutation = useMutation({
    mutationFn: profileApi.updateProfile,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['profile'] });
    },
  });

  if (isLoading) {
    return <div className="profile-loading">Loading profile...</div>;
  }

  if (error) {
    return <div className="profile-error">Failed to load profile</div>;
  }

  return (
    <div className="profile-page">
      <h1>Your Profile</h1>

      <section className="profile-section">
        <h2>Avatar</h2>
        <AvatarUpload
          currentUrl={profile?.avatarUrl}
          onUpload={(url) => updateMutation.mutate({ avatarUrl: url })}
        />
      </section>

      <section className="profile-section">
        <h2>Profile Information</h2>
        <ProfileForm
          profile={profile}
          onSave={(data) => updateMutation.mutate(data)}
          isSubmitting={updateMutation.isPending}
        />
      </section>

      <section className="profile-section">
        <h2>Preferences</h2>
        <PreferencesPanel
          preferences={profile?.preferences || {}}
          onSave={(prefs) => updateMutation.mutate({ preferences: prefs })}
        />
      </section>
    </div>
  );
};
```

**File**: `/workspace/my-project/src/components/Profile/ProfileForm.tsx`

```typescript
import React from 'react';
import { useForm } from 'react-hook-form';
import { UserProfile } from '../../types/profile';

interface ProfileFormProps {
  profile: UserProfile | null;
  onSave: (data: Partial<UserProfile>) => void;
  isSubmitting: boolean;
}

interface FormData {
  displayName: string;
  bio: string;
}

export const ProfileForm: React.FC<ProfileFormProps> = ({
  profile,
  onSave,
  isSubmitting,
}) => {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    defaultValues: {
      displayName: profile?.displayName || '',
      bio: profile?.bio || '',
    },
  });

  const onSubmit = (data: FormData) => {
    onSave(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="profile-form">
      <div className="form-group">
        <label htmlFor="displayName">Display Name</label>
        <input
          id="displayName"
          type="text"
          {...register('displayName', {
            required: 'Display name is required',
            maxLength: { value: 100, message: 'Max 100 characters' },
          })}
        />
        {errors.displayName && (
          <span className="error">{errors.displayName.message}</span>
        )}
      </div>

      <div className="form-group">
        <label htmlFor="bio">Bio</label>
        <textarea
          id="bio"
          rows={4}
          {...register('bio', {
            maxLength: { value: 500, message: 'Max 500 characters' },
          })}
        />
        {errors.bio && <span className="error">{errors.bio.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Saving...' : 'Save Profile'}
      </button>
    </form>
  );
};
```

### 3.3 Update CHANGELOG.md

**File**: `/workspace/my-project/CHANGELOG.md` (updated)

```markdown
# Changelog

## [Unreleased]

### Added
- User profile feature with complete CRUD functionality
- Database schema for user_profiles table with migrations
- REST API endpoints for profile management (/api/profile)
- React components: ProfilePage, ProfileForm, AvatarUpload, PreferencesPanel
- JWT-authenticated profile access

## [1.0.0] - 2026-01-15
...
```

### 3.4 Final Commit and Push

```bash
git add src/components/Profile/ CHANGELOG.md
git commit -m "Add React profile components and complete feature

- ProfilePage with query management
- ProfileForm with validation
- AvatarUpload and PreferencesPanel components
- Update CHANGELOG"

git push origin feature/user-profile
```

### 3.5 Create Pull Request

```bash
gh pr create \
  --title "Add user profile feature" \
  --body "## Summary
Complete implementation of user profile feature:
- Database schema and migrations
- REST API endpoints
- React UI components

## Testing
- Manual testing completed
- All endpoints verified

## Sessions
- session-profile-001: Database layer
- session-profile-002: API endpoints
- session-profile-003: UI components

Closes task-profile-001"
```

### 3.6 Final STATE.json

**File**: `/workspace/my-project/STATE.json` (at end of Session 3)

```json
{
  "sessionId": "session-profile-003",
  "status": "completed",
  "lastUpdated": "2026-02-02T12:00:00Z",
  "currentTask": null,
  "checkpoint": {
    "phase": "completed",
    "taskId": "task-profile-001",
    "completedSubtasks": ["sub-001", "sub-002", "sub-003"],
    "branch": "feature/user-profile",
    "commits": [
      {"hash": "abc1234", "message": "Add user profile database schema and migrations"},
      {"hash": "def5678", "message": "Add profile service and API endpoints"},
      {"hash": "ghi9012", "message": "Add React profile components and complete feature"}
    ],
    "prNumber": 45,
    "sessionHistory": ["session-profile-001", "session-profile-002", "session-profile-003"],
    "totalDuration": "5.5 hours across 3 sessions"
  },
  "errors": []
}
```

### 3.7 Final TODO.json

**File**: `/workspace/my-project/TODO.json` (at end of Session 3)

```json
{
  "version": "1.0.0",
  "lastUpdated": "2026-02-02T12:00:00Z",
  "tasks": [
    {
      "id": "task-profile-001",
      "title": "Implement user profile feature",
      "description": "Add complete user profile with avatar, settings, and preferences. Includes database schema, API endpoints, and React UI components.",
      "status": "completed",
      "priority": "high",
      "assignedSession": "session-profile-003",
      "estimatedSessions": 3,
      "actualSessions": 3,
      "createdAt": "2026-01-30T10:00:00Z",
      "updatedAt": "2026-02-02T12:00:00Z",
      "completedAt": "2026-02-02T12:00:00Z",
      "subtasks": [
        {"id": "sub-001", "title": "Database schema and migrations", "status": "completed"},
        {"id": "sub-002", "title": "API endpoints", "status": "completed"},
        {"id": "sub-003", "title": "React UI components", "status": "completed"}
      ],
      "sessionHistory": ["session-profile-001", "session-profile-002", "session-profile-003"],
      "prNumber": 45
    }
  ]
}
```

---

## STATE.json Evolution Summary

| Session | Status | Phase | Completed Subtasks | Key Changes |
|---------|--------|-------|-------------------|-------------|
| Start | completed | - | - | Previous session state |
| Session 1 Start | running | planning | - | New session, task claimed |
| Session 1 End | paused | development | sub-001 | DB schema done |
| Session 2 Start | running | development | sub-001 | Resumed from session 1 |
| Session 2 End | paused | development | sub-001, sub-002 | API done |
| Session 3 Start | running | development | sub-001, sub-002 | Resumed from session 2 |
| Session 3 End | completed | completed | sub-001, sub-002, sub-003 | Feature complete |

---

## Key Takeaways

1. **Checkpoint Preservation**: STATE.json captures enough context for seamless resume
2. **Incremental Progress**: Each session builds on previous work
3. **Clear Handoff**: Next steps documented in checkpoint
4. **History Tracking**: Session history maintained for audit
5. **Subtask Granularity**: Breaking work into subtasks enables partial completion
6. **Branch Continuity**: Same branch used across all sessions
