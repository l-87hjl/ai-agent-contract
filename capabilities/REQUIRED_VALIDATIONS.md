# Required Validations

This document specifies all validation checks that must pass before agent operations are executed. Validations are organized by operation type.

## Validation Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Before ANY Write                          │
├─────────────────────────────────────────────────────────────────┤
│  1. Path Allowlist Check                                        │
│  2. File Size Check                                            │
│  3. JSON Schema Validation (if applicable)                     │
│  4. Secret Detection Scan                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Before ANY Commit                          │
├─────────────────────────────────────────────────────────────────┤
│  5. Changelog Entry Exists (for main branch)                   │
│  6. STATE.json Updated                                         │
│  7. No Secrets in Staged Files                                 │
│  8. All Staged Files Pass Size Limits                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Before ANY PR                            │
├─────────────────────────────────────────────────────────────────┤
│  9. Branch Name Convention                                     │
│  10. Description Not Empty                                     │
│  11. Title Not Empty                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Pre-Write Validations

### V-001: Path Allowlist Check

**When**: Before any file write operation

**Purpose**: Ensure writes only occur to authorized locations

**Implementation**:
```python
def validate_path_allowlist(path: str) -> ValidationResult:
    # Resolve to absolute path
    absolute_path = os.path.abspath(path)

    # Check against blocked paths
    blocked_patterns = [
        r'\.git/',           # Git internals
        r'\.git$',           # .git directory itself
        r'node_modules/',    # Dependencies
        r'__pycache__/',     # Python cache
        r'\.env$',           # Environment files
        r'\.env\.',          # .env.local, .env.prod, etc.
        r'/agent-boot/',     # Boot repository
    ]

    for pattern in blocked_patterns:
        if re.search(pattern, absolute_path):
            return ValidationResult(
                valid=False,
                error=f"Path matches blocked pattern: {pattern}"
            )

    # Check path is within workspace
    if not absolute_path.startswith(workspace_root):
        return ValidationResult(
            valid=False,
            error="Path is outside workspace boundary"
        )

    return ValidationResult(valid=True)
```

**Error Response**:
```json
{
  "validation": "V-001",
  "valid": false,
  "error": "Path matches blocked pattern: \\.git/",
  "path": "/workspace/project/.git/config",
  "suggestion": "Use git commands to modify git configuration"
}
```

---

### V-002: File Size Check

**When**: Before any file write operation

**Purpose**: Prevent repository bloat and ensure reasonable file sizes

**Implementation**:
```python
def validate_file_size(content: bytes, path: str) -> ValidationResult:
    size_bytes = len(content)
    size_mb = size_bytes / (1024 * 1024)

    # Check absolute maximum
    if size_mb > 10:
        return ValidationResult(
            valid=False,
            error=f"File exceeds absolute maximum (10MB): {size_mb:.2f}MB"
        )

    # Check recommended maximum
    if size_mb > 1:
        # Allow with warning for non-binary files
        if is_binary(content):
            return ValidationResult(
                valid=False,
                error=f"Binary file exceeds limit (1MB): {size_mb:.2f}MB"
            )
        else:
            return ValidationResult(
                valid=True,
                warning=f"File is large ({size_mb:.2f}MB), consider splitting"
            )

    return ValidationResult(valid=True)
```

**Size Limits**:
| Category | Limit | Rationale |
|----------|-------|-----------|
| Text files | 1 MB recommended, 10 MB max | Large text files may indicate data that belongs in database |
| Binary files | 1 MB max | Use Git LFS for larger binaries |
| JSON/config | 1 MB max | Large configs should be split |
| Total per commit | 50 MB max | Prevents massive single commits |

**Error Response**:
```json
{
  "validation": "V-002",
  "valid": false,
  "error": "Binary file exceeds limit (1MB): 5.23MB",
  "path": "/workspace/project/assets/large-image.png",
  "suggestion": "Use Git LFS: git lfs track '*.png'"
}
```

---

### V-003: JSON Schema Validation

**When**: Before writing any file that has a registered schema

**Purpose**: Ensure data integrity and format compliance

**Schema Registry**:
| File Pattern | Schema Location |
|--------------|-----------------|
| `STATE.json` | `agent-boot/schemas/state.schema.json` |
| `TODO.json` | `agent-boot/schemas/todo.schema.json` |
| `AGENT_LINK.md` | `agent-boot/schemas/agent-link.schema.json` |
| `package.json` | Standard npm schema |
| `tsconfig.json` | Standard TypeScript schema |

**Implementation**:
```python
def validate_json_schema(content: str, path: str) -> ValidationResult:
    # Check if file has registered schema
    schema = get_schema_for_path(path)
    if schema is None:
        return ValidationResult(valid=True)  # No schema, pass

    # Parse JSON
    try:
        data = json.loads(content)
    except json.JSONDecodeError as e:
        return ValidationResult(
            valid=False,
            error=f"Invalid JSON: {e.msg} at line {e.lineno}"
        )

    # Validate against schema
    try:
        jsonschema.validate(data, schema)
    except jsonschema.ValidationError as e:
        return ValidationResult(
            valid=False,
            error=f"Schema validation failed: {e.message}",
            schema_path=e.schema_path,
            instance_path=e.absolute_path
        )

    return ValidationResult(valid=True)
```

**Error Response**:
```json
{
  "validation": "V-003",
  "valid": false,
  "error": "Schema validation failed: 'running' is not one of ['idle', 'paused', 'completed', 'failed']",
  "path": "/workspace/project/STATE.json",
  "schemaPath": ["properties", "status", "enum"],
  "instancePath": ["status"],
  "suggestion": "Use a valid status value: idle, paused, completed, or failed"
}
```

---

### V-004: Secret Detection Scan

**When**: Before any file write or commit

**Purpose**: Prevent accidental exposure of credentials

**Implementation**:
```python
def validate_no_secrets(content: str, path: str) -> ValidationResult:
    secret_patterns = [
        # API Keys
        (r'(?i)(api[_-]?key|apikey)\s*[=:]\s*["\']?[\w-]{20,}', 'API Key'),
        (r'(?i)(secret[_-]?key|secretkey)\s*[=:]\s*["\']?[\w-]{20,}', 'Secret Key'),

        # AWS
        (r'AKIA[0-9A-Z]{16}', 'AWS Access Key ID'),
        (r'(?i)aws[_-]?secret[_-]?access[_-]?key\s*[=:]\s*["\']?[\w/+=]{40}', 'AWS Secret'),

        # GitHub
        (r'ghp_[a-zA-Z0-9]{36}', 'GitHub Personal Access Token'),
        (r'github_pat_[a-zA-Z0-9]{22}_[a-zA-Z0-9]{59}', 'GitHub PAT (fine-grained)'),
        (r'gho_[a-zA-Z0-9]{36}', 'GitHub OAuth Token'),

        # Private Keys
        (r'-----BEGIN (?:RSA |EC |OPENSSH )?PRIVATE KEY-----', 'Private Key'),
        (r'-----BEGIN PGP PRIVATE KEY BLOCK-----', 'PGP Private Key'),

        # Database URLs with credentials
        (r'(?i)(mongodb|postgres|mysql|redis)://[^:]+:[^@]+@', 'Database URL with credentials'),

        # Generic secrets
        (r'(?i)(password|passwd|pwd)\s*[=:]\s*["\']?[^\s"\']{8,}', 'Password'),
        (r'(?i)bearer\s+[a-zA-Z0-9._-]{20,}', 'Bearer Token'),
    ]

    findings = []
    for pattern, secret_type in secret_patterns:
        matches = re.finditer(pattern, content)
        for match in matches:
            line_num = content[:match.start()].count('\n') + 1
            findings.append({
                'type': secret_type,
                'line': line_num,
                'preview': mask_secret(match.group())
            })

    if findings:
        return ValidationResult(
            valid=False,
            error=f"Potential secrets detected: {len(findings)} finding(s)",
            findings=findings
        )

    return ValidationResult(valid=True)
```

**Error Response**:
```json
{
  "validation": "V-004",
  "valid": false,
  "error": "Potential secrets detected: 2 finding(s)",
  "path": "/workspace/project/config.js",
  "findings": [
    {
      "type": "AWS Access Key ID",
      "line": 15,
      "preview": "AKIA****XXXX"
    },
    {
      "type": "Password",
      "line": 23,
      "preview": "password=****"
    }
  ],
  "suggestion": "Move secrets to environment variables or use a secrets manager"
}
```

---

## Pre-Commit Validations

### V-005: Changelog Entry Exists

**When**: Before committing to main branch

**Purpose**: Ensure all changes are documented

**Implementation**:
```python
def validate_changelog_entry(staged_files: List[str]) -> ValidationResult:
    # Check if CHANGELOG.md is being modified
    changelog_modified = 'CHANGELOG.md' in staged_files

    if not changelog_modified:
        return ValidationResult(
            valid=False,
            error="CHANGELOG.md must be updated for main branch commits"
        )

    # Verify entry has today's date or later
    changelog_content = read_staged_file('CHANGELOG.md')
    today = datetime.now().strftime('%Y-%m-%d')

    # Look for date pattern
    date_pattern = r'## \[.*?\] - (\d{4}-\d{2}-\d{2})'
    matches = re.findall(date_pattern, changelog_content)

    if not matches:
        return ValidationResult(
            valid=False,
            error="CHANGELOG.md has no dated entries"
        )

    latest_date = matches[0]
    if latest_date < today:
        return ValidationResult(
            valid=False,
            error=f"CHANGELOG.md latest entry ({latest_date}) is before today ({today})"
        )

    return ValidationResult(valid=True)
```

**Error Response**:
```json
{
  "validation": "V-005",
  "valid": false,
  "error": "CHANGELOG.md must be updated for main branch commits",
  "suggestion": "Add a changelog entry under today's date describing your changes"
}
```

---

### V-006: STATE.json Updated

**When**: Before any commit

**Purpose**: Maintain accurate session state

**Implementation**:
```python
def validate_state_updated(session_id: str) -> ValidationResult:
    state_path = os.path.join(workspace, 'STATE.json')

    if not os.path.exists(state_path):
        return ValidationResult(
            valid=False,
            error="STATE.json does not exist"
        )

    state = json.load(open(state_path))

    # Verify session ID matches
    if state.get('sessionId') != session_id:
        return ValidationResult(
            valid=False,
            error=f"STATE.json sessionId mismatch: expected {session_id}"
        )

    # Verify lastUpdated is recent (within last 5 minutes)
    last_updated = datetime.fromisoformat(state.get('lastUpdated', ''))
    if datetime.now() - last_updated > timedelta(minutes=5):
        return ValidationResult(
            valid=False,
            error="STATE.json lastUpdated is stale (>5 minutes old)"
        )

    return ValidationResult(valid=True)
```

**Error Response**:
```json
{
  "validation": "V-006",
  "valid": false,
  "error": "STATE.json lastUpdated is stale (>5 minutes old)",
  "suggestion": "Update STATE.json with current timestamp before committing"
}
```

---

### V-007: No Secrets in Staged Files

**When**: Before any commit

**Purpose**: Final check before code enters repository

**Implementation**:
```python
def validate_staged_files_no_secrets() -> ValidationResult:
    staged_files = git_get_staged_files()

    all_findings = []
    for file_path in staged_files:
        content = git_get_staged_content(file_path)
        result = validate_no_secrets(content, file_path)
        if not result.valid:
            all_findings.extend([
                {**f, 'file': file_path}
                for f in result.findings
            ])

    if all_findings:
        return ValidationResult(
            valid=False,
            error=f"Secrets detected in {len(all_findings)} location(s) across staged files",
            findings=all_findings
        )

    return ValidationResult(valid=True)
```

---

### V-008: Staged Files Size Limits

**When**: Before any commit

**Purpose**: Prevent oversized commits

**Implementation**:
```python
def validate_staged_files_size() -> ValidationResult:
    staged_files = git_get_staged_files()

    total_size = 0
    oversized = []

    for file_path in staged_files:
        content = git_get_staged_content(file_path)
        size = len(content)
        total_size += size

        if size > 1 * 1024 * 1024:  # 1MB
            oversized.append({
                'file': file_path,
                'size_mb': size / (1024 * 1024)
            })

    if oversized:
        return ValidationResult(
            valid=False,
            error=f"{len(oversized)} file(s) exceed size limit",
            oversized=oversized
        )

    if total_size > 50 * 1024 * 1024:  # 50MB total
        return ValidationResult(
            valid=False,
            error=f"Total commit size ({total_size / (1024*1024):.2f}MB) exceeds 50MB limit"
        )

    return ValidationResult(valid=True)
```

---

## Pre-PR Validations

### V-009: Branch Name Convention

**When**: Before creating a pull request

**Purpose**: Ensure consistent branch naming

**Implementation**:
```python
def validate_branch_name(branch: str) -> ValidationResult:
    # Pattern: prefix/description-with-dashes
    pattern = r'^(feature|fix|agent|update|hotfix|release)/[a-z0-9-]+$'

    if not re.match(pattern, branch):
        return ValidationResult(
            valid=False,
            error=f"Branch name '{branch}' does not follow convention",
            suggestion="Use format: prefix/description (e.g., feature/add-auth)"
        )

    if len(branch) > 100:
        return ValidationResult(
            valid=False,
            error=f"Branch name exceeds 100 characters ({len(branch)})"
        )

    # Check for protected branch names
    protected = ['main', 'master', 'develop', 'release', 'production']
    if branch in protected:
        return ValidationResult(
            valid=False,
            error=f"Cannot create PR from protected branch '{branch}'"
        )

    return ValidationResult(valid=True)
```

**Valid Branch Names**:
```
feature/add-user-authentication
fix/null-pointer-in-parser
agent/task-12345
update/dependency-versions
hotfix/security-patch
release/v2.0.0
```

**Invalid Branch Names**:
```
main                    # Protected
AddFeature              # Wrong case, no prefix
feature/Add_Feature     # Underscores and caps
my branch               # Spaces not allowed
feature/this-is-a-very-long-branch-name-that-exceeds-the-limit...
```

---

### V-010: PR Description Not Empty

**When**: Before creating a pull request

**Purpose**: Ensure PRs have meaningful descriptions

**Implementation**:
```python
def validate_pr_description(description: str) -> ValidationResult:
    # Remove whitespace and check length
    cleaned = description.strip()

    if not cleaned:
        return ValidationResult(
            valid=False,
            error="PR description cannot be empty"
        )

    if len(cleaned) < 20:
        return ValidationResult(
            valid=False,
            error="PR description too short (minimum 20 characters)",
            suggestion="Add more detail about what this PR does and why"
        )

    return ValidationResult(valid=True)
```

---

### V-011: PR Title Not Empty

**When**: Before creating a pull request

**Purpose**: Ensure PRs have meaningful titles

**Implementation**:
```python
def validate_pr_title(title: str) -> ValidationResult:
    cleaned = title.strip()

    if not cleaned:
        return ValidationResult(
            valid=False,
            error="PR title cannot be empty"
        )

    if len(cleaned) < 10:
        return ValidationResult(
            valid=False,
            error="PR title too short (minimum 10 characters)"
        )

    if len(cleaned) > 100:
        return ValidationResult(
            valid=False,
            error="PR title too long (maximum 100 characters)"
        )

    return ValidationResult(valid=True)
```

---

## Validation Execution Order

All validations run in a specific order. If any validation fails, subsequent validations are skipped and the operation is blocked.

### Write Operation Order
```
1. V-001: Path Allowlist Check
2. V-002: File Size Check
3. V-003: JSON Schema Validation
4. V-004: Secret Detection Scan
```

### Commit Operation Order
```
1. [All write validations for each staged file]
2. V-006: STATE.json Updated
3. V-007: No Secrets in Staged Files
4. V-008: Staged Files Size Limits
5. V-005: Changelog Entry (main branch only)
```

### PR Operation Order
```
1. V-009: Branch Name Convention
2. V-010: PR Description Not Empty
3. V-011: PR Title Not Empty
```

---

## Validation Result Format

All validations return a consistent result format:

```json
{
  "validation": "V-001",
  "valid": true|false,
  "error": "Error message if invalid",
  "warning": "Warning message if valid but concerning",
  "suggestion": "How to fix the issue",
  "details": { }
}
```

This format is logged to the audit trail for every validation check.
