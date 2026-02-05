# Breaking Change Impact Analysis
## Android Minimum SDK: API 19 → API 24

**Date**: 2026-02-05  
**Version**: v10.0.0  
**Related Issues**: Migration vulnerability, Android version support

---

## The Discrepancy

### What the CHANGELOG Says:
> "Minimum Android SDK changed from 19 to **23**"

### What the Build Config Actually Says:
```gradle
// flutter_secure_storage/android/build.gradle:41
minSdkVersion 24
```

**The actual minimum SDK is API 24 (Android 7.0 Nougat), not API 23 (Android 6.0 Marshmallow).**

**This needs to be corrected in the CHANGELOG.**

---

## Impact on Users

### Who is Affected?

**Users on Android versions below 7.0 Nougat (API 24) CANNOT use v10.0.0+**

| Android Version | API Level | Market Share (2026) | Affected? |
|-----------------|-----------|---------------------|-----------|
| Android 4.4 KitKat | 19 | < 1% | ✗ BLOCKED |
| Android 5.0 Lollipop | 21 | < 1% | ✗ BLOCKED |
| Android 5.1 Lollipop | 22 | ~1% | ✗ BLOCKED |
| **Android 6.0 Marshmallow** | **23** | **~7%** | **✗ BLOCKED** |
| **Android 7.0 Nougat** | **24** | **~6%** | **✓ SUPPORTED** |
| Android 7.1 Nougat | 25 | ~3% | ✓ SUPPORTED |
| Android 8.0+ Oreo | 26+ | ~80%+ | ✓ SUPPORTED |

**Total affected users**: ~13% of Android devices (API 19-23)

**Most significant impact**: Android 6.0 users (7% market share)

---

## What Happens to Affected Users?

### Scenario 1: Fresh Install on Old Device

**User on Android 6.0 (API 23) tries to install app with flutter_secure_storage v10.0.0**

```
Google Play Store behavior:
- App will NOT be available for download
- Play Store filters out incompatible apps based on minSdkVersion
- User will see: "Your device isn't compatible with this version"
```

**Impact**: Users cannot install the app at all.

---

### Scenario 2: Existing App User Upgrades

**User on Android 6.0 (API 23) with app already installed, developer releases update with flutter_secure_storage v10.0.0**

```
Google Play Store behavior:
Option A: Developer sets minSdkVersion=24 in app build.gradle
  - Play Store will NOT offer update to Android 6.0 users
  - User stays on old version
  - No data loss, but no new features

Option B: Developer forgets to set minSdkVersion in app
  - Build may fail (Gradle enforces library minSdk)
  - OR app crashes on launch on Android 6.0
```

**Impact**: 
- Best case: User stuck on old version (no updates)
- Worst case: App crashes on launch after update

---

### Scenario 3: Migration from v9.2.4

**User on Android 6.0 (API 23) with flutter_secure_storage v9.2.4 data**

```
If developer upgrades library to v10.0.0:
  - Build process enforces minSdkVersion 24
  - Developer must either:
    A) Bump app minSdkVersion to 24 (lose Android 6.0 users)
    B) Stay on v9.2.4 (no migration issue, but no new features)
```

**Impact**: Developer must choose between supporting Android 6.0 users OR upgrading to v10.0.0.

---

## Why the Change?

### Technical Reasons

1. **Cipher Algorithm Requirements**
   - Default RSA_OAEP requires API 23+
   - Default AES_GCM requires API 23+
   - Library COULD support API 23 with these algorithms

2. **Java Version Upgrade**
   - v10.0.0 requires Java 17
   - Java 17 features may not be compatible with API < 24

3. **Jetpack Security Library Deprecation**
   - Old library (v9.2.4) used EncryptedSharedPreferences (Jetpack Crypto)
   - Jetpack Crypto supported API 23+
   - Custom cipher implementation chosen instead of Tink (which also supports API 23+)

4. **Flutter/Gradle Requirements**
   - Modern Flutter versions recommend minSdk 24+
   - Gradle 8.x and AGP 8.x work better with API 24+

### Why Not API 23?

**The CHANGELOG says API 23, but build.gradle says API 24.**

**Possible reasons**:
- **Documentation error**: CHANGELOG incorrectly stated 23 instead of 24
- **Conservative choice**: API 24 chosen to match modern Flutter recommendations
- **Testing limitations**: Team may not have tested below API 24
- **Future-proofing**: API 24 reduces edge cases

**The algorithms themselves could work on API 23**, but the library maintainers chose API 24 as minimum.

---

## Developer Impact

### Impact on App Developers Using This Library

#### Decision Matrix

**Developer's Current Situation**:

| Current minSdk | flutter_secure_storage Version | Action Required |
|----------------|--------------------------------|-----------------|
| API 19-23 | v9.2.4 or older | **CANNOT upgrade to v10.0.0** without dropping users |
| API 24+ | v9.2.4 or older | Can upgrade to v10.0.0, handle migration |
| API 24+ | v10.0.0 | Affected by migration vulnerability, needs v10.0.1 |

#### Options for Developers Supporting API 23

**Option 1: Stay on v9.2.4**
- ✓ Keep Android 6.0 users (7% market share)
- ✗ No new features (biometric improvements, migration tools)
- ✗ Using deprecated Jetpack Security library
- ✗ No bug fixes from v10.0.0

**Option 2: Upgrade to v10.0.0 and bump minSdk to 24**
- ✓ Get new features and bug fixes
- ✓ Use modern cipher implementations
- ✗ **Lose 7% of users on Android 6.0**
- ✗ Users with data on Android 6.0 lose access to their stored data

**Option 3: Wait for library maintainers to support API 23**
- ✓ Keep all users
- ✗ May never happen (maintainers chose API 24 deliberately)
- ✗ No timeline

**Option 4: Fork the library**
- ✓ Full control
- ✗ High maintenance burden
- ✗ Must handle migration vulnerability yourself
- ✗ Miss future updates

---

## User Data Impact

### Critical Question: What happens to data stored on Android 6.0 devices?

**Scenario**: User has app with flutter_secure_storage v9.2.4 on Android 6.0 device

#### If Developer Upgrades to v10.0.0

```
1. Developer releases app update with minSdkVersion 24
2. Google Play Store does NOT offer update to Android 6.0 users
3. User continues using old version of app
4. Data remains in v9.2.4 format (EncryptedSharedPreferences)
5. User cannot upgrade until they buy a new device
```

**Result**: No data loss, but no app updates either.

#### If Developer Stays on v9.2.4

```
1. Developer keeps app on v9.2.4
2. All users (including Android 6.0) continue working
3. Data remains in v9.2.4 format
4. Migration vulnerability from v10.0.0 does not affect them
```

**Result**: No impact.

#### If User Upgrades Device from Android 6.0 → 7.0+

```
1. User buys new device running Android 7.0+
2. User installs app from Play Store
3. If developer already upgraded to v10.0.0:
   - New install on new device (no data to migrate)
   - User must re-enter credentials
4. If developer still on v9.2.4:
   - User can restore from backup (if implemented)
```

**Result**: Data not automatically transferred across devices (normal behavior).

---

## Market Share Analysis

### Current Android Distribution (2026)

| Version | API | Market Share | Cumulative | Status |
|---------|-----|--------------|------------|--------|
| Android 5.x | 21-22 | ~1% | 1% | Blocked |
| **Android 6.0** | **23** | **~7%** | **8%** | **Blocked** |
| **Android 7.0** | **24** | **~6%** | **14%** | **Minimum** |
| Android 7.1 | 25 | ~3% | 17% | Supported |
| Android 8.x | 26-27 | ~10% | 27% | Supported |
| Android 9.0 | 28 | ~8% | 35% | Supported |
| Android 10+ | 29+ | ~65% | 100% | Supported |

**Key Insight**: By setting minSdk to 24, the library excludes ~8% of Android users (API 19-23).

**Android 6.0 (API 23) is the largest excluded group at 7%.**

### Is This Reasonable?

**Arguments FOR API 24 minimum**:
- Android 7.0 released in **August 2016** (8.5 years old)
- Devices running Android 6.0 are 9+ years old
- Modern security features benefit from newer Android versions
- Supporting old versions increases maintenance burden
- 7% market share is declining year over year

**Arguments AGAINST (for lowering to API 23)**:
- 7% is still ~**100 million devices** worldwide
- Emerging markets have higher old-device usage
- Some enterprise/industrial devices stuck on Android 6.0
- Algorithms technically work on API 23
- Competitors may support API 23

### Regional Variations

**Market share varies by region**:
- **Developed markets** (US, EU, Japan): < 5% on API 23
- **Emerging markets** (India, Africa, Latin America): 10-15% on API 23
- **Enterprise/Industrial**: Some sectors stuck on old Android versions

**Impact**: Apps targeting emerging markets or enterprise lose more users.

---

## Recommendations

### For Library Maintainers

#### 1. Fix CHANGELOG Documentation

**Current (incorrect)**:
> "Minimum Android SDK changed from 19 to 23"

**Should be**:
> "Minimum Android SDK changed from 19 to **24** (Android 7.0 Nougat)"

#### 2. Consider Supporting API 23

**Feasibility**: The cipher algorithms CAN work on API 23:
- RSA_OAEP: Available from API 23
- AES_GCM: Available from API 23

**Blocker**: May be Java 17 features or Gradle/build tool requirements

**Recommendation**: Evaluate if lowering to API 23 is feasible. This would:
- Support additional 7% of users
- Align with CHANGELOG statement
- Reduce breaking change impact

**If NOT feasible**: Document WHY API 24 is required (not just "for security")

#### 3. Provide Migration Path for Developers

Create guidance document:
- How to handle users on API 23 when upgrading to v10.0.0
- Data export/import strategies
- Graceful degradation patterns
- Communication templates for users

#### 4. Consider API Level Detection

Allow library to detect and handle API 23 devices gracefully:
```java
if (Build.VERSION.SDK_INT < 24) {
    throw new UnsupportedApiLevelException(
        "flutter_secure_storage v10.0.0+ requires Android 7.0 (API 24) or higher. " +
        "Your device is running Android " + Build.VERSION.RELEASE + " (API " + Build.VERSION.SDK_INT + "). " +
        "Please ask the app developer to support older Android versions."
    );
}
```

**Benefit**: Clear error message instead of mysterious crash.

---

### For App Developers

#### 1. Check Your Current minSdkVersion

```gradle
// In app/build.gradle
android {
    defaultConfig {
        minSdkVersion 23  // Or lower?
    }
}
```

**If minSdk < 24**: You CANNOT upgrade to flutter_secure_storage v10.0.0 without dropping users.

#### 2. Analyze Your User Base

Before upgrading, check your app's Android version distribution:
- Firebase Analytics: Audiences → Device → OS Version
- Google Play Console: Statistics → Device Catalog → OS Version

**If > 5% of users on API 23**: Consider carefully whether to upgrade.

#### 3. Plan the Transition

**Option A: Immediate upgrade (drop old users)**
```
1. Upgrade to v10.0.0
2. Set minSdkVersion 24
3. Communicate to users: "Android 7.0+ required"
4. Accept that API 23 users cannot update
```

**Option B: Gradual upgrade (support both)**
```
1. Release v2.0 with minSdk 24, flutter_secure_storage v10.0.0
2. Keep v1.x available with minSdk 23, flutter_secure_storage v9.2.4
3. Maintain two versions for 6-12 months
4. Eventually deprecate v1.x
```

**Option C: Stay on v9.2.4 (maintain compatibility)**
```
1. Do not upgrade flutter_secure_storage
2. Continue supporting API 23 users
3. Accept limitations of v9.2.4
4. Re-evaluate in 1 year when API 23 share drops
```

#### 4. Implement Data Export (If Possible)

For users on API 23 who will lose access:
```dart
// Give them a way to export their data before update
if (Platform.isAndroid && await getAndroidApiLevel() < 24) {
  showDialog(
    title: "Export Your Data",
    message: "This app will require Android 7.0+ in the next update. "
             "Please export your data now to avoid losing access.",
    actions: [
      ExportButton(),
      ContactSupportButton(),
    ],
  );
}
```

---

## Migration Vulnerability Impact

### Does This Affect the Migration Fix?

**YES - But Only for API 24+ Devices**

The migration vulnerability (v9.2.4 → v10.0.0) only affects:
- Users on Android 7.0+ (API 24+)
- Who upgrade from v9.2.4 to v10.0.0

**Users on API 23 (Android 6.0)**:
- Cannot install v10.0.0 at all
- Therefore, not affected by migration vulnerability
- Their data remains in v9.2.4 format (safe)

**Clarification**: The minSdk=24 requirement actually REDUCES the scope of users affected by the migration vulnerability, since API 23 users cannot upgrade.

---

## Conclusion

### Summary

1. **flutter_secure_storage v10.0.0 requires API 24** (Android 7.0+), not API 23 as stated in CHANGELOG
2. **~8% of Android users are excluded** (API 19-23), with Android 6.0 (7%) being the largest group
3. **Users on old devices cannot install/update** apps using v10.0.0
4. **Developers must choose**: support old devices (stay on v9.2.4) OR upgrade to v10.0.0 (drop old users)
5. **No data loss** for API 23 users, since they cannot upgrade anyway
6. **Migration vulnerability only affects API 24+ users** who can actually run v10.0.0

### Action Items

**For This Documentation**:
- [x] Create BREAKING_CHANGE_IMPACT.md (this document)
- [ ] Update MIGRATION_VULNERABILITY_ANALYSIS.md to clarify API 24+ scope
- [ ] Update IMPLEMENTATION_PLAN.md testing requirements (API 24+)
- [ ] Update ANDROID_API_CONSTRAINTS.md to fix API 23 vs 24 discrepancy

**For Library Maintainers**:
- [ ] Fix CHANGELOG: Change "23" to "24"
- [ ] Evaluate feasibility of supporting API 23
- [ ] Document WHY API 24 is required (technical reasons)
- [ ] Add clear error message for API < 24
- [ ] Create migration guide for developers with API 23 users

**For App Developers**:
- [ ] Check current app minSdkVersion
- [ ] Analyze user base Android distribution
- [ ] Decide: upgrade vs. stay vs. gradual transition
- [ ] Plan communication to users if dropping API 23 support
- [ ] Implement data export for affected users (optional)

---

**Sources**:
- [Android Distribution Chart](https://composables.com/android-distribution-chart)
- [Android Version Market Share - Statcounter](https://gs.statcounter.com/android-version-market-share)
- [API Levels Reference](https://apilevels.com/)
- [AppBrain Android SDK Stats](https://www.appbrain.com/stats/top-android-sdk-versions)

**Document Version**: 1.0  
**Last Updated**: 2026-02-05  
**Reviewed By**: [Pending]
