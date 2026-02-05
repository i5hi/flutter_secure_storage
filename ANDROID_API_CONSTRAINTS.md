# Android API Level and Device Constraints
## flutter_secure_storage v10.0.0+

**Document Version**: 1.0  
**Last Updated**: 2026-02-05  
**Related to**: Migration Safety Fix (v10.0.1)

---

## Minimum Android API Requirements

### Version Changes

| Version | Min SDK | Target SDK | Compile SDK | Java Version |
|---------|---------|------------|-------------|--------------|
| **v9.2.4** | API 19 (Android 4.4 KitKat) | 34 | 34 | Java 8 |
| **v10.0.0** | API 24 (Android 7.0 Nougat) | 36 | 36 | Java 17 |

### Why the Change?

**v10.0.0 raised minimum SDK from 19 → 24 because**:
1. Custom cipher implementations require newer Android APIs
2. Biometric authentication APIs require API 23+
3. Modern KeyStore features require API 23+
4. Google deprecated Jetpack Security library (which supported lower APIs)
5. Security improvements in Android 7.0+ (better KeyStore, hardware-backed keys)

---

## API Level Constraints by Feature

### 1. Core Storage (Non-Biometric)

**Minimum**: API 24 (Android 7.0 Nougat)

**Supported Algorithms**:
- `RSA_ECB_PKCS1Padding` - API 1+ (but min SDK is 24)
- `RSA_ECB_OAEPwithSHA_256andMGF1Padding` (default) - API 23+ (Android 6.0 Marshmallow)
- `AES_CBC_PKCS7Padding` - API 1+ (but min SDK is 24)
- `AES_GCM_NoPadding` (default) - API 23+ (Android 6.0 Marshmallow)

**File**: `KeyCipherAlgorithm.java` and `StorageCipherAlgorithm.java`

```java
// Key Cipher Algorithms
RSA_ECB_PKCS1Padding(KeyCipherImplementationRSA18::new, 1)
RSA_ECB_OAEPwithSHA_256andMGF1Padding(KeyCipherImplementationRSAOAEP::new, Build.VERSION_CODES.M) // API 23
AES_GCM_NoPadding(KeyCipherImplementationAES23::new, Build.VERSION_CODES.M) // API 23

// Storage Cipher Algorithms
AES_CBC_PKCS7Padding(StorageCipherImplementationAES18::new, 1)
AES_GCM_NoPadding(null, Build.VERSION_CODES.M) // API 23
```

**Automatic Fallback**: If user requests an algorithm not supported by device API level, the library automatically falls back to default algorithms.

**File**: `StorageCipherFactory.java:44-48`
```java
currentStorageAlgorithm = (currentStorageAlgorithmTmp.minVersionCode <= Build.VERSION.SDK_INT) 
    ? currentStorageAlgorithmTmp 
    : DEFAULT_STORAGE_ALGORITHM;
```

---

### 2. Biometric Authentication

**Minimum for Biometric Features**: API 28 (Android 9.0 Pie)

**API Level Requirements**:
- **API 23-27**: No biometric support, graceful degradation to non-authenticated cipher
- **API 28-29**: BiometricPrompt available, limited authenticator options
- **API 30+ (Android 11+)**: Full BiometricManager support with BIOMETRIC_STRONG + DEVICE_CREDENTIAL

**File**: `FlutterSecureStorage.java:282, 946, 974, 992, 1051, 1064`

#### Key Code Sections:

**Biometric Check (API 28 minimum)**:
```java
// FlutterSecureStorage.java:282
if (Build.VERSION.SDK_INT < Build.VERSION_CODES.P) {
    // Android version below API 28 (Android 9.0)
    // No biometric authentication available
    // Use non-authenticated cipher
}
```

**Biometric Availability Check**:
```java
// FlutterSecureStorage.java:946
public boolean isBiometricAvailable() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        // Android 11+ (API 30): Use BiometricManager
        BiometricManager biometricManager = context.getSystemService(BiometricManager.class);
        int result = biometricManager.canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG | 
            BiometricManager.Authenticators.DEVICE_CREDENTIAL
        );
        return result == BiometricManager.BIOMETRIC_SUCCESS;
    } else {
        // API 28-29: Fallback check
        // ...
    }
}
```

**Biometric Enforcement Check**:
```java
// FlutterSecureStorage.java:974
private void ensureBiometricAvailable(boolean enforceRequired) throws Exception {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.P) {
        if (enforceRequired) {
            throw new Exception("BIOMETRIC_UNAVAILABLE: Biometric authentication requires Android 9 (API 28) or higher");
        }
        // If not enforced, gracefully degrade to non-biometric
        return;
    }
    // ... check device capabilities
}
```

**BiometricPrompt Configuration**:
```java
// FlutterSecureStorage.java:1064
BiometricPrompt.PromptInfo.Builder promptInfoBuilder = new BiometricPrompt.PromptInfo.Builder()
    .setTitle(config.getPrefOptionBiometricPromptTitle())
    .setSubtitle(config.getPrefOptionBiometricPromptSubtitle());

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
    // API 30+: Allow both biometric and device credential
    promptInfoBuilder.setAllowedAuthenticators(
        BiometricManager.Authenticators.BIOMETRIC_STRONG | 
        BiometricManager.Authenticators.DEVICE_CREDENTIAL
    );
} else {
    // API 28-29: Must use setNegativeButtonText
    promptInfoBuilder.setNegativeButtonText("Cancel");
}
```

**Graceful Degradation**:
- `enforceBiometrics: false` (default) → Falls back to non-authenticated cipher on API < 28
- `enforceBiometrics: true` → Throws error on API < 28

---

### 3. StrongBox Support

**Minimum**: API 28 (Android 9.0 Pie)

**Feature**: Hardware-backed keystore in dedicated secure element (Titan M chip on Pixel devices)

**File**: `MasterKey.java:116, 227, 303`

```java
// MasterKey.java:116
public boolean isStrongBoxBacked() {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.P || mKeyGenParameterSpec == null) {
        return false;
    }
    // Check if key uses StrongBox
}

// MasterKey.java:303
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P && builder.mRequestStrongBoxBacked) {
    if (builder.mContext.getPackageManager().hasSystemFeature(
        PackageManager.FEATURE_STRONGBOX_KEYSTORE)) {
        keyGenBuilder.setIsStrongBoxBacked(true);
    }
}
```

**Notes**:
- StrongBox is optional, not all devices support it
- Library gracefully degrades to regular KeyStore if unavailable
- Provides enhanced security on supported devices (Pixel 3+, Samsung S9+)

---

### 4. User Authentication Parameters

**Minimum**: API 30 (Android 11)

**Feature**: Fine-grained control over authentication timeout and type

**File**: `MasterKey.java:294-296`

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
    Api30Impl.setUserAuthenticationParameters(
        keyGenBuilder,
        builder.mUserAuthenticationValidityDurationSeconds,
        KeyProperties.AUTH_BIOMETRIC_STRONG | KeyProperties.AUTH_DEVICE_CREDENTIAL
    );
} else {
    // API < 30: Use deprecated setUserAuthenticationValidityDurationSeconds
    keyGenBuilder.setUserAuthenticationValidityDurationSeconds(
        builder.mUserAuthenticationValidityDurationSeconds
    );
}
```

**Impact**: API 28-29 have limited authentication control compared to API 30+

---

### 5. RSA Cipher Provider

**API-specific implementations**:

**File**: `KeyCipherImplementationRSA18.java:108-110`

```java
protected Cipher getRSACipher() throws Exception {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.M) {
        // Android 5.x and below: Use AndroidOpenSSL provider
        return Cipher.getInstance("RSA/ECB/PKCS1Padding", "AndroidOpenSSL");
    } else {
        // Android 6.0+: Use AndroidKeyStoreBCWorkaround provider
        return Cipher.getInstance("RSA/ECB/PKCS1Padding", "AndroidKeyStoreBCWorkaround");
    }
}
```

**Note**: Since v10.0.0 min SDK is 24, the `< M` branch is never executed but remains for compatibility.

---

## Device-Specific Constraints

### 1. Biometric Hardware

**Requirement**: Device must have biometric hardware (fingerprint, face, iris)

**Graceful Degradation**:
- If `enforceBiometrics: false` → Library falls back to non-biometric cipher
- If `enforceBiometrics: true` → Library throws error if biometric unavailable

**Check**: `BiometricManager.canAuthenticate()` result codes:
- `BIOMETRIC_SUCCESS` → Biometric available
- `BIOMETRIC_ERROR_NO_HARDWARE` → No biometric sensor
- `BIOMETRIC_ERROR_HW_UNAVAILABLE` → Sensor temporarily unavailable
- `BIOMETRIC_ERROR_NONE_ENROLLED` → No biometric credentials enrolled
- `BIOMETRIC_ERROR_SECURITY_UPDATE_REQUIRED` → Update required

---

### 2. Device Security (PIN/Pattern/Password)

**Requirement**: For biometric authentication to work, device must have at least one of:
- PIN
- Pattern
- Password
- Biometric (fingerprint, face, iris)

**Graceful Degradation**:
```java
// FlutterSecureStorage.java:281-284
if (cipher == null 
    || Build.VERSION.SDK_INT < Build.VERSION_CODES.P 
    || (!enforceRequired && !deviceHasSecurity)) {
    // No biometric authentication needed - use non-authenticated cipher
}
```

**Impact**: Devices without any security setup can still use flutter_secure_storage with non-biometric ciphers.

---

### 3. KeyStore Availability

**Requirement**: Android KeyStore must be available and functional

**Potential Issues**:
- KeyStore corruption (device-specific bug)
- KeyStore locked after boot (requires device unlock first)
- KeyStore full (rare, but possible on old/low-storage devices)

**Migration Impact**: If KeyStore is unavailable, both old (EncryptedSharedPreferences) and new (custom ciphers) storage will fail.

---

### 4. StrongBox Support

**Supported Devices** (non-exhaustive):
- Google Pixel 3, 3 XL, 3a, 4, 5, 6, 7, 8 (Titan M/M2 chip)
- Samsung Galaxy S9, S10, S20, S21, S22, S23 series
- Samsung Galaxy Note 10, 20 series

**Unsupported Devices**: Most budget/mid-range devices

**Library Behavior**: Requests StrongBox if available, falls back to regular KeyStore if not.

---

## Migration Constraints (v9.2.4 → v10.0.0)

### 1. EncryptedSharedPreferences Availability

**v9.2.4 Dependency**:
```gradle
implementation 'androidx.security:security-crypto:1.1.0-alpha06'
implementation 'com.google.crypto.tink:tink-android:1.9.0'
```

**v10.0.0 Dependency**:
```gradle
implementation("com.google.crypto.tink:tink-android:1.20.0")
```

**Migration Constraint**: For migration to work, `EncryptedSharedPreferences` must be initializable:
- MasterKey must exist in KeyStore
- Tink library must be available
- SharedPreferences file must not be corrupted

**File**: `FlutterSecureStorage.java:173-174`

```java
try {
    SharedPreferences encryptedPreferences = initializeEncryptedSharedPreferencesManager(context);
    // ... attempt migration
} catch (Exception e) {
    Log.e(TAG, "EncryptedSharedPreferences initialization failed. Falling back to custom ciphers.", e);
    // Migration skipped, data becomes invisible
}
```

**Risk**: If ESP initialization fails, data remains in old storage but becomes inaccessible.

---

### 2. Algorithm Compatibility

**Device must support target algorithms**:

| Algorithm | Min API | Fallback |
|-----------|---------|----------|
| RSA_OAEP (default) | API 23 | RSA_PKCS1 |
| AES_GCM (default) | API 23 | AES_CBC |

**File**: `StorageCipherFactory.java:44-48`

**Automatic Fallback**: Library automatically selects compatible algorithm if requested algorithm not supported.

---

### 3. Memory Constraints

**Migration Process**:
1. Read all data from EncryptedSharedPreferences (plaintext in memory)
2. Encrypt all data with new cipher (in-memory)
3. Write all data to new storage

**Memory Usage**: For large datasets (100+ keys), memory usage can be significant:
- Plaintext: ~N bytes (where N = total data size)
- Encrypted: ~N * 1.5 bytes (base64 encoding overhead)
- Total peak memory: ~2.5N bytes

**Recommendation**: For datasets > 1MB, consider implementing incremental migration in Phase 2 improvements.

---

### 4. Storage Space

**Migration creates temporary duplicate data**:
- Old storage: EncryptedSharedPreferences (Tink-encrypted)
- New storage: Custom cipher SharedPreferences (AES-GCM encrypted)

**Peak storage usage**: 2x data size during migration

**Cleanup**: Old data deleted after successful migration

---

## Testing Requirements for Migration Fix

### API Level Coverage

**Minimum test matrix**:

| API Level | Version | Device Example | Priority |
|-----------|---------|----------------|----------|
| API 24 | Android 7.0 Nougat | Emulator | P0 (min SDK) |
| API 26 | Android 8.0 Oreo | Samsung Galaxy S8 | P1 |
| API 28 | Android 9.0 Pie | Pixel 3 | P0 (biometric baseline) |
| API 29 | Android 10 | Pixel 4 | P1 |
| API 30 | Android 11 | Pixel 5 | P0 (full biometric) |
| API 33 | Android 13 | Pixel 7 | P1 |
| API 34 | Android 14 | Pixel 8 | P0 (current stable) |
| API 36 | Android 15 (future) | Emulator | P2 |

### Device Constraint Testing

**Required test scenarios**:

1. **No Biometric Hardware**
   - Emulator without biometric
   - Old devices (pre-2017)

2. **No Device Security**
   - Fresh device with no PIN/pattern/password set
   - Test graceful degradation

3. **KeyStore Locked**
   - Device just booted, not unlocked yet
   - Test "StrongBox unavailable" error

4. **Low Storage**
   - Device with < 100MB free space
   - Test migration failure handling

5. **KeyStore Corruption**
   - Simulate KeyStore failure
   - Test fallback to EncryptedSharedPreferences

6. **StrongBox Devices**
   - Pixel 3+ or Samsung S9+
   - Test StrongBox usage

---

## Migration Fix Implementation Considerations

### 1. API Level Checks in Atomic Migration

**Recommendation**: Add API level logging in migration process

```java
private MigrationResult migrateFromEncryptedSharedPreferences(...) {
    Log.i(TAG, "Migration starting on API " + Build.VERSION.SDK_INT);
    Log.i(TAG, "Device: " + Build.MANUFACTURER + " " + Build.MODEL);
    Log.i(TAG, "Target algorithms: " + currentKeyAlgorithm + " / " + currentStorageAlgorithm);
    
    // ... migration logic
}
```

**Benefit**: Helps diagnose device-specific migration failures.

---

### 2. Graceful Degradation in Recovery API

**Recommendation**: Recovery API should detect API level constraints

```java
// In getMigrationStatus()
Map<String, Object> status = new HashMap<>();
status.put("apiLevel", Build.VERSION.SDK_INT);
status.put("biometricAvailable", isBiometricAvailable());
status.put("deviceSecure", hasDeviceSecurity());
status.put("canUseBiometric", Build.VERSION.SDK_INT >= Build.VERSION_CODES.P);
```

**Benefit**: User can understand if migration failure is due to device constraints.

---

### 3. Biometric Migration Edge Cases

**Scenario**: User has biometric data in v9.2.4, but device no longer supports biometric in v10.0.0

**Current behavior**: Migration may fail if biometric prompt cannot be shown

**Recommendation**: Add biometric availability check before biometric migration

```java
if (fromBiometric || toBiometric) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.P) {
        Log.e(TAG, "Biometric migration not possible on API < 28");
        return MigrationResult.failure("Biometric requires API 28+", null);
    }
    if (!isBiometricAvailable()) {
        Log.e(TAG, "Biometric migration not possible: hardware unavailable");
        return MigrationResult.failure("Biometric unavailable", null);
    }
    // ... proceed with biometric migration
}
```

---

## Summary of Constraints for Contributor

### Absolute Requirements

1. **Minimum SDK**: API 24 (Android 7.0 Nougat)
   - Cannot go lower without major refactoring
   - All custom ciphers require API 23+ features

2. **Biometric Features**: API 28+ (Android 9.0 Pie)
   - BiometricPrompt API introduced in API 28
   - Graceful degradation on API 24-27

3. **Full Biometric Features**: API 30+ (Android 11)
   - BiometricManager with fine-grained control
   - BIOMETRIC_STRONG + DEVICE_CREDENTIAL support

### Device Requirements

1. **KeyStore**: Must be available and functional
2. **Storage**: Sufficient space for duplicate data during migration (2x data size)
3. **Memory**: Sufficient RAM for in-memory migration (2.5x data size)
4. **Biometric** (optional): Hardware + enrollment (API 28+)
5. **Device Security** (optional): PIN/pattern/password for biometric fallback

### Migration-Specific Constraints

1. **EncryptedSharedPreferences**: Must be initializable for migration
2. **Tink Library**: Version compatibility (1.9.0 → 1.20.0)
3. **Algorithm Support**: Device must support target cipher algorithms
4. **No Data Loss Tolerance**: Atomic migration critical for user trust

### Testing Requirements

1. **API Coverage**: API 24, 28, 30, 34 minimum
2. **Device Types**: With/without biometric, with/without device security
3. **Storage States**: Low storage, corrupted KeyStore, locked KeyStore
4. **Manufacturer Diversity**: Test on Samsung, Pixel, generic emulator

---

## Recommendations for Implementation

### 1. Add API Level Validation

```java
private void validateApiLevelSupport() {
    if (Build.VERSION.SDK_INT < 24) {
        throw new Exception("flutter_secure_storage requires Android 7.0 (API 24) or higher");
    }
    
    if (config.shouldUseBiometric() && Build.VERSION.SDK_INT < Build.VERSION_CODES.P) {
        if (config.shouldEnforceBiometric()) {
            throw new Exception("Biometric authentication requires Android 9.0 (API 28) or higher");
        } else {
            Log.w(TAG, "Biometric requested but API < 28. Falling back to non-biometric cipher.");
        }
    }
}
```

### 2. Add Device Capability Checks

```java
private Map<String, Boolean> getDeviceCapabilities() {
    Map<String, Boolean> caps = new HashMap<>();
    caps.put("biometricHardware", isBiometricAvailable());
    caps.put("deviceSecure", hasDeviceSecurity());
    caps.put("strongBoxSupported", isStrongBoxSupported());
    caps.put("keyStoreAvailable", isKeyStoreAvailable());
    return caps;
}
```

### 3. Enhanced Error Messages

Include API level and device info in error messages:

```java
String errorMessage = String.format(
    "Migration failed: %s (API %d, %s %s)", 
    error.getMessage(),
    Build.VERSION.SDK_INT,
    Build.MANUFACTURER,
    Build.MODEL
);
```

---

## Further Reading

- [Android KeyStore System](https://developer.android.com/privacy-and-security/keystore)
- [BiometricPrompt API](https://developer.android.com/reference/android/hardware/biometrics/BiometricPrompt)
- [StrongBox](https://developer.android.com/privacy-and-security/keystore#HardwareSecurityModule)
- [Android Version Distribution](https://developer.android.com/about/dashboards)

---

**Document prepared by**: Migration Safety Task Force  
**For questions**: See MIGRATION_FIX_SUMMARY.md
