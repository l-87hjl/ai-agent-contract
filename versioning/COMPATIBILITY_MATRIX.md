# Compatibility Matrix

This document defines which versions of the agent contract work with which versions of agent-boot, and provides guidance for handling version mismatches.

## Version Compatibility Table

| Contract Version | Boot Version | Status | Notes |
|-----------------|--------------|--------|-------|
| 1.0.0 | 1.0.0 | ✅ Compatible | Initial release |
| 1.0.0 | 1.0.1 | ✅ Compatible | Bug fixes only |
| 1.0.0 | 1.0.2 | ✅ Compatible | Bug fixes only |
| 1.0.0 | 1.0.x | ✅ Compatible | All 1.0.x patch versions |
| 1.0.0 | 1.1.0 | ⚠️ Forward Compatible | New boot features ignored |
| 1.0.0 | 2.0.0 | ❌ Incompatible | Breaking changes |

### Legend

| Symbol | Meaning |
|--------|---------|
| ✅ | Fully compatible, all features work |
| ⚠️ | Forward compatible, contract works but may not use new features |
| ❌ | Incompatible, do not use together |

---

## Version Compatibility Rules

### Semantic Versioning

Both contract and boot follow semantic versioning (MAJOR.MINOR.PATCH):

- **MAJOR**: Breaking changes, incompatible
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, fully compatible

### Compatibility Logic

```python
def check_compatibility(contract_version: str, boot_version: str) -> str:
    contract = parse_semver(contract_version)
    boot = parse_semver(boot_version)

    # Major version must match
    if contract.major != boot.major:
        return "incompatible"

    # Contract minor can be <= boot minor (boot has more features)
    if contract.minor > boot.minor:
        return "incompatible"

    # Patch versions are always compatible within same major.minor
    return "compatible"
```

---

## Handling Version Mismatches

### Scenario 1: Boot Version Newer (Minor)

**Example**: Contract 1.0.0 with Boot 1.1.0

**Behavior**:
- Agent operates normally with contract 1.0.0 rules
- New boot 1.1.0 features are available but not used
- No action required

**Verification**:
```bash
# Check versions at session start
CONTRACT_VERSION=$(jq -r '.contractVersion' ai-agent-contract/versioning/CONTRACT_VERSION.json)
BOOT_VERSION=$(jq -r '.version' agent-boot/package.json)

echo "Contract: $CONTRACT_VERSION"
echo "Boot: $BOOT_VERSION"
```

### Scenario 2: Boot Version Newer (Patch)

**Example**: Contract 1.0.0 with Boot 1.0.3

**Behavior**:
- Fully compatible
- Boot may include bug fixes
- No action required

### Scenario 3: Boot Version Older (Minor)

**Example**: Contract 1.1.0 with Boot 1.0.0

**Behavior**:
- **Warning**: Contract may reference features boot doesn't have
- Agent should check for missing schemas
- Graceful degradation if possible

**Recovery**:
```bash
# Upgrade boot to compatible version
cd agent-boot
git fetch origin
git checkout v1.1.0  # or latest compatible
```

### Scenario 4: Major Version Mismatch

**Example**: Contract 1.0.0 with Boot 2.0.0 (or vice versa)

**Behavior**:
- **Error**: Incompatible versions
- Agent must not start
- User intervention required

**Error Message**:
```
ERROR: Version incompatibility detected

Contract version: 1.0.0
Boot version: 2.0.0

These versions are incompatible. Major version mismatch indicates
breaking changes that prevent safe operation.

To resolve:
1. Upgrade contract to 2.x: Update ai-agent-contract repository
2. Or downgrade boot to 1.x: git checkout v1.0.x in agent-boot

See COMPATIBILITY_MATRIX.md for compatible version pairs.
```

---

## Upgrade Paths

### Upgrading Contract Version

When a new contract version is released:

1. **Review Changelog**
   ```bash
   jq '.changelog' versioning/CONTRACT_VERSION.json
   ```

2. **Check Breaking Changes**
   - If `breakingChanges: true`, review impact
   - Update workspace configurations if needed

3. **Update Repository**
   ```bash
   cd ai-agent-contract
   git fetch origin
   git checkout v1.1.0  # New version
   ```

4. **Verify Compatibility**
   ```bash
   # Ensure boot version is compatible
   REQUIRED_BOOT=$(jq -r '.minimumBootVersion' versioning/CONTRACT_VERSION.json)
   echo "Minimum boot version required: $REQUIRED_BOOT"
   ```

5. **Test in Non-Production**
   - Run agent in test workspace first
   - Verify all operations work correctly

### Upgrading Boot Version

When upgrading agent-boot:

1. **Check Contract Compatibility**
   ```bash
   # Get compatible contract versions from boot
   jq '.compatibleContractVersions' agent-boot/package.json
   ```

2. **Update Boot Repository**
   ```bash
   cd agent-boot
   git fetch origin
   git checkout v1.0.3  # New version
   ```

3. **Verify Schemas**
   ```bash
   # Ensure all required schemas exist
   ls agent-boot/schemas/
   ```

4. **Run Validation**
   ```bash
   # Validate existing STATE.json against new schemas
   npx ajv validate -s agent-boot/schemas/state.schema.json -d workspace/STATE.json
   ```

---

## Version Detection

### At Session Start

```python
def verify_versions():
    # Load contract version
    contract_version = load_json('ai-agent-contract/versioning/CONTRACT_VERSION.json')

    # Load boot version
    boot_package = load_json('agent-boot/package.json')
    boot_version = boot_package['version']

    # Check compatibility
    compatibility = check_compatibility(
        contract_version['contractVersion'],
        boot_version
    )

    if compatibility == "incompatible":
        raise VersionMismatchError(
            contract=contract_version['contractVersion'],
            boot=boot_version
        )

    if compatibility == "forward_compatible":
        log_warning(f"Boot {boot_version} has features not in contract {contract_version['contractVersion']}")

    log_info(f"Versions compatible: Contract {contract_version['contractVersion']}, Boot {boot_version}")
```

### Version Check Endpoint

For programmatic verification:

```json
GET /agent/version

Response:
{
  "contract": {
    "version": "1.0.0",
    "effectiveDate": "2026-02-01"
  },
  "boot": {
    "version": "1.0.2"
  },
  "compatibility": "compatible",
  "warnings": []
}
```

---

## Future Version Planning

### Planned Versions

| Version | Planned Date | Changes |
|---------|--------------|---------|
| 1.0.1 | 2026-03-01 | Bug fixes, documentation updates |
| 1.1.0 | 2026-06-01 | New capabilities (TBD) |
| 2.0.0 | TBD | Major refactoring (if needed) |

### Deprecation Policy

1. Features are deprecated in minor versions
2. Deprecated features removed in next major version
3. Minimum 6 months notice before removal
4. Migration guide provided for breaking changes

### Version Lifecycle

```
1.0.0 ──► 1.0.1 ──► 1.0.2 ──► ... (patch: bug fixes)
  │
  └──► 1.1.0 ──► 1.1.1 ──► ... (minor: new features)
         │
         └──► 2.0.0 ──► ... (major: breaking changes)
```

---

## Compatibility Testing

### Automated Tests

```bash
#!/bin/bash
# test-compatibility.sh

CONTRACT_VERSIONS=("1.0.0")
BOOT_VERSIONS=("1.0.0" "1.0.1" "1.0.2")

for contract in "${CONTRACT_VERSIONS[@]}"; do
  for boot in "${BOOT_VERSIONS[@]}"; do
    echo "Testing: Contract $contract with Boot $boot"

    # Checkout versions
    git -C ai-agent-contract checkout "v$contract"
    git -C agent-boot checkout "v$boot"

    # Run compatibility check
    ./check-compatibility.sh

    # Run integration tests
    ./run-integration-tests.sh

    echo "Result: $?"
  done
done
```

### Manual Verification Checklist

- [ ] Contract version loaded correctly
- [ ] Boot version loaded correctly
- [ ] Compatibility check passes
- [ ] All schemas validate
- [ ] All validations run
- [ ] Audit trail writes correctly
- [ ] Session starts and ends normally
