# Implementation Plan: Migration Safety Fix
## flutter_secure_storage v10.0.1

**Date**: 2026-02-05  
**Priority**: P0 - CRITICAL  
**Target Release**: v10.0.1

---

## Overview

This plan addresses critical migration vulnerabilities identified in v10.0.0 that can cause data loss when upgrading from v9.2.4. The implementation focuses on making migration atomic, adding rollback capability, and providing recovery mechanisms.

---

## Goals

### Primary Goals (MUST HAVE)

1. **Zero data loss**: Migration must never lose user data
2. **Atomic operations**: All-or-nothing migration (no partial states)
3. **Rollback capability**: Revert to source on failure
4. **Recovery API**: Allow users to retry failed migrations
5. **State tracking**: Monitor migration progress and failures

### Secondary Goals (SHOULD HAVE)

6. **Clear warnings**: Alert users about breaking changes
7. **Comprehensive testing**: Cover all failure scenarios
8. **User documentation**: Guide users through migration
9. **Telemetry**: Track migration success rates

---

## Technical Implementation

### Phase 1: Core Safety (P0 - Critical)

#### Task 1.1: Atomic Migration with Rollback

**File**: `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStorage.java`

**Location**: Lines 1117-1137 (replace `migrateFromEncryptedSharedPreferences` method)

**Changes**:
1. Change return type from `void` to `MigrationResult`
2. Add `MigrationResult` inner class with success/failure states
3. Implement 4-phase migration:
   - Phase 1: Read all data from source (no modifications)
   - Phase 2: Encrypt all data in-memory (no storage writes)
   - Phase 3: Write all data atomically using `commit()` (not `apply()`)
   - Phase 4: Verify writes, then delete from source
4. Use `commit()` instead of `apply()` for synchronous verification
5. Catch exceptions at each phase and return `MigrationResult.failure()`
6. Only delete from source after ALL writes verified

**Code structure**:
```java
private MigrationResult migrateFromEncryptedSharedPreferences(
    SharedPreferences source, 
    SharedPreferences target
) {
    Map<String, String> plaintextCache = new HashMap<>();
    Map<String, String> encryptedCache = new HashMap<>();
    
    try {
        // Phase 1: Read
        for (Map.Entry<String, ?> entry : source.getAll().entrySet()) {
            // Store in plaintextCache
        }
        
        // Phase 2: Encrypt
        for (Map.Entry<String, String> entry : plaintextCache.entrySet()) {
            // Encrypt and store in encryptedCache
            // If ANY encryption fails → return MigrationResult.failure()
        }
        
        // Phase 3: Write atomically
        SharedPreferences.Editor editor = target.edit();
        for (Map.Entry<String, String> entry : encryptedCache.entrySet()) {
            editor.putString(entry.getKey(), entry.getValue());
        }
        boolean writeSuccess = editor.commit();  // Synchronous!
        if (!writeSuccess) {
            return MigrationResult.failure("Write failed", null);
        }
        
        // Phase 4: Verify and cleanup
        for (String key : encryptedCache.keySet()) {
            if (!target.contains(key)) {
                return MigrationResult.failure("Verification failed", null);
            }
        }
        
        // All verified, safe to delete from source
        SharedPreferences.Editor sourceEditor = source.edit();
        for (String key : plaintextCache.keySet()) {
            sourceEditor.remove(key);
        }
        sourceEditor.commit();
        
        return MigrationResult.success(encryptedCache.size());
        
    } catch (Exception e) {
        return MigrationResult.failure("Migration error", e);
    }
}

private static class MigrationResult {
    final boolean success;
    final int migratedCount;
    final String errorMessage;
    final Exception exception;
    
    static MigrationResult success(int count) { /* ... */ }
    static MigrationResult failure(String msg, Exception ex) { /* ... */ }
}
```

**Testing**:
- Unit test: Successful migration
- Unit test: Encryption failure (verify source data intact)
- Unit test: Write failure (verify source data intact)
- Unit test: Verification failure (verify source data intact)
- Integration test: Full flow with real EncryptedSharedPreferences

**Success criteria**:
- [ ] All tests passing
- [ ] Code review approved
- [ ] Manual testing on 3+ devices
- [ ] No data loss in any failure scenario

---

#### Task 1.2: Migration State Tracking

**File**: `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStorage.java`

**Location**: Add new methods (after line 820)

**Changes**:
1. Add constants for state tracking keys
2. Create `MigrationState` enum (NOT_STARTED, IN_PROGRESS, COMPLETED, FAILED_RETRYABLE, FAILED_PERMANENT)
3. Implement `recordMigrationAttempt()` method
4. Implement `getMigrationState()` method
5. Implement `shouldRetryMigration()` method (max 3 retries)
6. Store: state, attempt count, last error, timestamp

**Code structure**:
```java
private static final String PREF_MIGRATION_STATE = "MIGRATION_STATE";
private static final String PREF_MIGRATION_ATTEMPT_COUNT = "MIGRATION_ATTEMPT_COUNT";
private static final String PREF_MIGRATION_LAST_ERROR = "MIGRATION_LAST_ERROR";
private static final String PREF_MIGRATION_LAST_ATTEMPT_TIMESTAMP = "MIGRATION_LAST_ATTEMPT";

private enum MigrationState {
    NOT_STARTED,
    IN_PROGRESS,
    COMPLETED,
    FAILED_RETRYABLE,
    FAILED_PERMANENT
}

private void recordMigrationAttempt(
    SharedPreferences configSource, 
    MigrationState state, 
    String errorMessage
) {
    // Increment counter, store state, timestamp, error
}

private boolean shouldRetryMigration(SharedPreferences configSource) {
    // Check state and attempt count
    // Return false if COMPLETED or FAILED_PERMANENT
    // Return false if attempts >= 3 (mark as FAILED_PERMANENT)
    // Return true otherwise
}
```

**Testing**:
- Unit test: State transitions
- Unit test: Retry limits (3 max)
- Unit test: Permanent failure after 3 attempts
- Integration test: State persistence across app restarts

**Success criteria**:
- [ ] State tracking works across app restarts
- [ ] Max retry limit enforced
- [ ] Clear error messages stored

---

#### Task 1.3: Safe Migration Wrapper

**File**: `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStorage.java`

**Location**: Lines 171-250 (refactor `initialize()` method)

**Changes**:
1. Check migration state before attempting migration
2. Respect retry limits (max 3 attempts)
3. Handle `FAILED_PERMANENT` state
4. Update migration wrapper to use new `MigrationResult` return type
5. Record state transitions at each step
6. Provide clear error messages for permanent failures

**Code flow**:
```java
if (!isAlreadyMigrated) {
    // Check if we should retry
    if (!shouldRetryMigration(configSource)) {
        MigrationState state = getMigrationState(configSource);
        if (state == COMPLETED) {
            // Already done, continue
        } else {
            // FAILED_PERMANENT - give user options
            String lastError = configSource.getString(PREF_MIGRATION_LAST_ERROR, "Unknown");
            callback.onError(new Exception(
                "Migration failed permanently: " + lastError + 
                ". Options: 1) Set encryptedSharedPreferences=true, or 2) Set resetOnError=true to delete."
            ));
            return;
        }
    }
    
    // Mark as IN_PROGRESS
    recordMigrationAttempt(configSource, MigrationState.IN_PROGRESS, null);
    
    // Attempt migration
    initializeStorageCipher(configSource, new SecurePreferencesCallback<>() {
        @Override
        public void onSuccess(Void unused) {
            MigrationResult result = migrateFromEncryptedSharedPreferences(esp, target);
            
            if (result.success) {
                // SUCCESS
                preferences = target;
                recordMigrationAttempt(configSource, MigrationState.COMPLETED, null);
                setEncryptedPrefsMigrated(configSource);
                callback.onSuccess(null);
            } else {
                // FAILURE - but data is safe
                recordMigrationAttempt(configSource, MigrationState.FAILED_RETRYABLE, 
                                      result.errorMessage);
                preferences = esp;  // Fall back
                callback.onSuccess(null);  // Don't fail the app
            }
        }
        
        @Override
        public void onError(Exception e) {
            recordMigrationAttempt(configSource, MigrationState.FAILED_RETRYABLE, 
                                  "Cipher init: " + e.getMessage());
            preferences = esp;  // Fall back
            callback.onSuccess(null);
        }
    });
}
```

**Testing**:
- Integration test: First migration success
- Integration test: First migration failure, second success
- Integration test: 3 failures → permanent failure
- Integration test: Permanent failure with encryptedSharedPreferences=true fallback

**Success criteria**:
- [ ] Migration retries work correctly
- [ ] Permanent failure handled gracefully
- [ ] Clear error messages to user
- [ ] Data never lost

---

#### Task 1.4: Recovery API

**File**: `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStoragePlugin.java`

**Location**: Add new method handlers (after line 100)

**Changes**:
1. Add method channel constants: `getMigrationStatus`, `retryMigration`, `forceUseEncryptedSharedPreferences`
2. Implement `getMigrationStatus()` - returns Map with state, attempts, lastError, timestamp
3. Implement `retryMigration()` - resets state and triggers re-initialization
4. Update method dispatcher to route new methods

**Code structure**:
```java
private static final String METHOD_GET_MIGRATION_STATUS = "getMigrationStatus";
private static final String METHOD_RETRY_MIGRATION = "retryMigration";

@Override
public void onMethodCall(@NonNull MethodCall call, @NonNull Result result) {
    // ... existing methods ...
    
    switch (call.method) {
        case METHOD_GET_MIGRATION_STATUS:
            getMigrationStatus(result);
            break;
        case METHOD_RETRY_MIGRATION:
            retryMigration(result);
            break;
        // ...
    }
}

private void getMigrationStatus(Result result) {
    Map<String, Object> status = new HashMap<>();
    status.put("state", configSource.getString("MIGRATION_STATE", "NOT_STARTED"));
    status.put("attempts", configSource.getInt("MIGRATION_ATTEMPT_COUNT", 0));
    status.put("lastError", configSource.getString("MIGRATION_LAST_ERROR", null));
    status.put("lastAttempt", configSource.getLong("MIGRATION_LAST_ATTEMPT", 0));
    result.success(status);
}

private void retryMigration(Result result) {
    // Reset migration state
    SharedPreferences.Editor editor = configSource.edit();
    editor.putString("MIGRATION_STATE", "NOT_STARTED");
    editor.putInt("MIGRATION_ATTEMPT_COUNT", 0);
    editor.commit();
    
    // Trigger re-initialization
    // ...
}
```

**Dart API** (add to `flutter_secure_storage/lib/flutter_secure_storage.dart`):
```dart
Future<Map<String, dynamic>> getMigrationStatus() async {
  return await _platform.getMigrationStatus();
}

Future<void> retryMigration() async {
  return await _platform.retryMigration();
}
```

**Testing**:
- Unit test: Status API returns correct data
- Unit test: Retry API resets state correctly
- Integration test: Full recovery flow (fail → status → retry → success)

**Success criteria**:
- [ ] API accessible from Dart
- [ ] Status accurately reflects migration state
- [ ] Retry successfully triggers re-migration
- [ ] Documentation updated

---

#### Task 1.5: resetOnError Warning

**File**: `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStorageConfig.java`

**Location**: Constructor (around line 30)

**Changes**:
1. Detect if user explicitly set `resetOnError` option
2. If not set and defaults to `true`, log prominent warning
3. Warning should be visible in Logcat with clear formatting

**Code**:
```java
public FlutterSecureStorageConfig(Map<String, String> options) {
    // ... existing code ...
    
    boolean hasExplicitResetOnError = options.containsKey(PREF_OPTION_DELETE_ON_FAILURE);
    boolean resetOnError = getBooleanOption(options, PREF_OPTION_DELETE_ON_FAILURE, true);
    
    if (!hasExplicitResetOnError && resetOnError) {
        Log.w(TAG, "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
        Log.w(TAG, "⚠️  IMPORTANT: resetOnError defaults to TRUE in v10.0.0");
        Log.w(TAG, "⚠️  BREAKING CHANGE from v9.2.4 (default was false)");
        Log.w(TAG, "⚠️  If migration fails, ALL DATA WILL BE DELETED");
        Log.w(TAG, "⚠️  Set resetOnError=false to preserve data on errors");
        Log.w(TAG, "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━");
    }
}
```

**Testing**:
- Manual test: Verify warning appears in Logcat
- Manual test: Verify warning doesn't appear when explicitly set

**Success criteria**:
- [ ] Warning clearly visible
- [ ] Warning only shows when using default
- [ ] Warning text actionable

---

### Phase 2: Testing & Validation (P0 - Critical)

#### Task 2.1: Unit Test Suite

**File**: Create `flutter_secure_storage/android/src/test/java/com/it_nomads/fluttersecurestorage/MigrationTest.java`

**Test cases**:

1. `testMigration_Success_AllKeysTransferred()`
   - Setup ESP with 3 keys
   - Migrate
   - Verify all 3 in target, none in source

2. `testMigration_EncryptionFailure_NoDataLost()`
   - Mock cipher to fail on encryption
   - Verify migration returns failure
   - Verify data still in source

3. `testMigration_WriteFailure_NoDataLost()`
   - Mock SharedPreferences.commit() to return false
   - Verify migration returns failure
   - Verify data still in source

4. `testMigration_PartialEncryptionFailure_Rollback()`
   - Mock cipher to fail on 3rd of 5 keys
   - Verify migration returns failure
   - Verify ALL keys still in source

5. `testMigration_VerificationFailure_NoCleanup()`
   - Mock target.contains() to return false
   - Verify migration returns failure
   - Verify data still in source

6. `testMigration_RetryAfterFailure_Succeeds()`
   - First attempt fails
   - Second attempt succeeds
   - Verify data migrated

7. `testMigration_MaxRetriesExceeded_PermanentFailure()`
   - Fail 3 times
   - Verify state becomes FAILED_PERMANENT
   - Verify shouldRetryMigration() returns false

8. `testMigrationState_Transitions()`
   - Verify state transitions: NOT_STARTED → IN_PROGRESS → COMPLETED
   - Verify state transitions: NOT_STARTED → IN_PROGRESS → FAILED_RETRYABLE

9. `testMigrationState_PersistsAcrossRestarts()`
   - Set state to COMPLETED
   - Create new FlutterSecureStorage instance
   - Verify state still COMPLETED

10. `testRecoveryAPI_GetStatus()`
    - Verify status API returns correct data

11. `testRecoveryAPI_RetryMigration()`
    - Set state to FAILED_RETRYABLE
    - Call retryMigration()
    - Verify state reset to NOT_STARTED

**Success criteria**:
- [ ] All tests passing
- [ ] 100% code coverage for migration logic
- [ ] Tests run in CI/CD

---

#### Task 2.2: Integration Test Suite

**File**: Create `flutter_secure_storage/example/integration_test/migration_test.dart`

**Test scenarios**:

1. Fresh install (no migration needed)
2. Upgrade from v9.2.4 with data (successful migration)
3. Migration failure with resetOnError=false (fallback to ESP)
4. Migration failure with resetOnError=true (data deleted)
5. Migration retry after failure
6. Permanent failure after 3 attempts

**Success criteria**:
- [ ] All integration tests passing
- [ ] Tests run on real devices (not just emulator)
- [ ] Tests cover Android API levels 23-34

---

### Phase 3: Documentation (P1 - High)

#### Task 3.1: Migration Guide

**File**: Create `flutter_secure_storage/MIGRATION_GUIDE_v10.md`

**Contents**:
1. Overview of changes
2. Breaking changes (resetOnError default)
3. Migration process explanation
4. Troubleshooting guide
5. Recovery procedures
6. FAQ

**Success criteria**:
- [ ] Guide reviewed by 2+ team members
- [ ] Tested by following guide on fresh project
- [ ] Linked from main README

---

#### Task 3.2: API Documentation

**File**: `flutter_secure_storage/lib/flutter_secure_storage.dart`

**Changes**:
1. Add dartdoc comments for `getMigrationStatus()`
2. Add dartdoc comments for `retryMigration()`
3. Add usage examples
4. Document return types and exceptions

**Success criteria**:
- [ ] Dartdoc generated correctly
- [ ] Examples runnable

---

#### Task 3.3: CHANGELOG Update

**File**: `flutter_secure_storage/CHANGELOG.md`

**Changes**:
1. Add v10.0.1 section at top
2. Highlight critical bug fix
3. Document breaking change warning
4. Link to migration guide
5. Acknowledge affected users

**Success criteria**:
- [ ] Clear and prominent
- [ ] Actionable guidance
- [ ] Apologetic tone for issue

---

### Phase 4: Release Preparation (P1 - High)

#### Task 4.1: Beta Release

**Version**: v10.0.1-beta.1

**Checklist**:
- [ ] All P0 tasks completed
- [ ] All tests passing
- [ ] Code review approved
- [ ] Beta release notes published
- [ ] Announced in GitHub discussions
- [ ] Monitor for 7 days

---

#### Task 4.2: Stable Release

**Version**: v10.0.1

**Checklist**:
- [ ] Beta feedback incorporated
- [ ] No critical issues reported
- [ ] Documentation complete
- [ ] Release notes finalized
- [ ] pub.dev published
- [ ] GitHub release created
- [ ] Announcement posted

---

## Timeline

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| **Phase 1**: Core Safety | 3-4 days | None |
| Task 1.1: Atomic migration | 1 day | None |
| Task 1.2: State tracking | 0.5 days | None |
| Task 1.3: Safe wrapper | 0.5 days | 1.1, 1.2 |
| Task 1.4: Recovery API | 1 day | 1.2 |
| Task 1.5: Warning | 0.5 days | None |
| **Phase 2**: Testing | 2-3 days | Phase 1 |
| Task 2.1: Unit tests | 1.5 days | 1.1, 1.2, 1.3 |
| Task 2.2: Integration tests | 1.5 days | 1.1, 1.2, 1.3 |
| **Phase 3**: Documentation | 1-2 days | Phase 1 |
| Task 3.1: Migration guide | 1 day | Phase 1 |
| Task 3.2: API docs | 0.5 days | 1.4 |
| Task 3.3: CHANGELOG | 0.5 days | Phase 1 |
| **Phase 4**: Release | 10-14 days | All above |
| Task 4.1: Beta release | 7 days | All above |
| Task 4.2: Stable release | 3 days | Beta success |

**Total estimated time**: 16-23 days (3-4 weeks)

**Fastest path**: 16 days (assuming no blockers)

---

## Risk Assessment

### High Risks

1. **Breaking existing apps**
   - Risk: Changes to migration logic might break apps that worked (albeit incorrectly)
   - Mitigation: Extensive testing, beta release, monitor closely

2. **Incomplete test coverage**
   - Risk: Edge cases not caught by tests
   - Mitigation: Comprehensive test plan, manual testing, dogfooding

3. **Performance regression**
   - Risk: Atomic migration might be slower
   - Mitigation: Performance testing, optimize if needed

### Medium Risks

4. **API changes**
   - Risk: New methods might conflict with user code
   - Mitigation: Choose non-conflicting names, document clearly

5. **Documentation insufficient**
   - Risk: Users don't understand how to recover
   - Mitigation: Clear examples, FAQ, support channels

### Low Risks

6. **Beta adoption**
   - Risk: Not enough beta testers
   - Mitigation: Announce widely, incentivize testing

---

## Success Criteria

### Must Have (P0)

- [ ] Zero data loss in all tested scenarios
- [ ] Migration is atomic (all-or-nothing)
- [ ] Failed migrations can be retried
- [ ] Users can check migration status
- [ ] Clear error messages for failures
- [ ] All unit tests passing
- [ ] All integration tests passing

### Should Have (P1)

- [ ] Migration guide published
- [ ] API documentation complete
- [ ] Beta release successful (no critical bugs)
- [ ] User feedback positive

### Nice to Have (P2)

- [ ] Telemetry for migration success rates
- [ ] Performance metrics collected
- [ ] Automated migration health checks

---

## Rollback Plan

If critical issues discovered after release:

1. **Immediate**: Publish rollback documentation
   - How to downgrade to v9.2.4
   - How to manually migrate data
   - How to recover from failed state

2. **Short-term**: Publish hotfix release
   - Disable problematic feature
   - Restore v9.2.4 behavior as option
   - Communicate clearly with users

3. **Long-term**: Re-evaluate approach
   - Gather more requirements
   - Design alternative solution
   - Plan v10.0.2 or v10.1.0

---

## Communication Plan

### Internal

- Daily standups on progress
- Slack channel for coordination
- Weekly status reports

### External

- GitHub issue for tracking: `#[NUMBER] - Migration Safety Fix`
- Beta announcement in GitHub Discussions
- Stable release announcement
- Blog post explaining the issue and fix
- Social media updates

---

## Resources Required

### Personnel

- 1 Senior Android Engineer (lead)
- 1 Flutter Engineer (API + Dart side)
- 1 QA Engineer (testing)
- 1 Technical Writer (documentation)

### Infrastructure

- CI/CD pipeline for automated testing
- Multiple test devices (API 23-34)
- Beta testing group (20+ users)

### Time

- 3-4 weeks full-time effort
- 1 week beta testing period
- 1 week buffer for issues

---

## Post-Release

### Monitoring

- Track GitHub issues for migration-related bugs
- Monitor pub.dev analytics for adoption rate
- Collect user feedback via survey
- Track crash reports for migration failures

### Maintenance

- Respond to issues within 24 hours
- Publish hotfixes if critical bugs found
- Update documentation based on feedback
- Plan future improvements

---

## Lessons Learned

(To be filled after release)

- What went well?
- What could be improved?
- What surprised us?
- What would we do differently?

---

## Appendix

### A. File Locations

| Component | File Path |
|-----------|-----------|
| Main Logic | `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStorage.java` |
| Plugin | `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStoragePlugin.java` |
| Config | `flutter_secure_storage/android/src/main/java/com/it_nomads/fluttersecurestorage/FlutterSecureStorageConfig.java` |
| Unit Tests | `flutter_secure_storage/android/src/test/java/com/it_nomads/fluttersecurestorage/MigrationTest.java` |
| Integration Tests | `flutter_secure_storage/example/integration_test/migration_test.dart` |
| Migration Guide | `flutter_secure_storage/MIGRATION_GUIDE_v10.md` |

### B. Method Signatures

```java
// FlutterSecureStorage.java
private MigrationResult migrateFromEncryptedSharedPreferences(SharedPreferences source, SharedPreferences target)
private void recordMigrationAttempt(SharedPreferences configSource, MigrationState state, String errorMessage)
private MigrationState getMigrationState(SharedPreferences configSource)
private boolean shouldRetryMigration(SharedPreferences configSource)

// FlutterSecureStoragePlugin.java
private void getMigrationStatus(Result result)
private void retryMigration(Result result)
```

### C. Configuration Keys

```java
// SharedPreferences keys
ENCRYPTED_PREFERENCES_MIGRATED = "ENCRYPTED_PREFERENCES_MIGRATED"
PREF_MIGRATION_STATE = "MIGRATION_STATE"
PREF_MIGRATION_ATTEMPT_COUNT = "MIGRATION_ATTEMPT_COUNT"
PREF_MIGRATION_LAST_ERROR = "MIGRATION_LAST_ERROR"
PREF_MIGRATION_LAST_ATTEMPT_TIMESTAMP = "MIGRATION_LAST_ATTEMPT"
```

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-05  
**Next Review**: After beta release
