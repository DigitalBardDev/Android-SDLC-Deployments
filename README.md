# Android Release Engineering & Production Deployment Pipeline

An end-to-end Android Software Development Lifecycle (SDLC) and release management specification covering build automation, App Bundle compilation, cryptographic signing, and Google Play Store deployment pipelines.

---

# PART 1: Overview & Infrastructure

## The Deployment Architecture

This repository documents the production release pipelines for three distinct Android applications: **Branching Realms**, **Scratchboard Art Studio**, and **Homophones**. These projects represent two distinct mobile compilation ecosystems:

1. **Native Kotlin / Modern Android Stack (`BranchingRealms`)**: A native application built with Jetpack Compose, Kotlin Coroutines, Dagger Hilt, and Room, managed by Android Gradle Plugin (AGP) 8.9.3.
2. **Game Engine Native Export (`scratchboard-art`)**: A 2D/3D graphics application built on Godot Engine 4.6 (Mobile Vulkan renderer), compiling through a custom Godot Android Gradle template (AGP 8.6.1) that binds C++ engine libraries and Play Asset Delivery modules.
3. **Cryptographic Key Management Pipeline (`Homophones`)**: An isolated release signing infrastructure maintaining production keystores and signing identity controls separated from active application codebases.

The deployment architecture standardizes build automation, target SDK alignment, permission scoping, and store ingestion pipelines across both native JVM and native C++/NDK targets.

```
+-----------------------------------------------------------------------------------+
|                            Source Code Repositories                               |
|                                                                                   |
|  +---------------------------+  +------------------------+  +------------------+  |
|  |      BranchingRealms      |  |    scratchboard-art    |  |    Homophones    |  |
|  | (Native Kotlin + Compose) |  |   (Godot 4.6 Engine)   |  | (Isolated Keys)  |  |
|  +-------------+-------------+  +-----------+------------+  +--------+---------+  |
+----------------|----------------------------|------------------------|------------+
                 |                            |                        |
                 v                            v                        |
+----------------------------------------------------------+           |
|                 Compilation & Build Engine               |           |
|                                                          |           |
|  - AGP 8.9.3 / Kotlin 2.0.0    - AGP 8.6.1 / NDK 28      |           |
|  - KSP Schema Processing       - Play Asset Delivery     |           |
|  - Compose BOM Integration     - C++ ABI Filtering       |           |
|  - R8 Code & Resource Shrink   - Resource Optimization   |           |
+-----------------------------+----------------------------+           |
                              |                                        |
                              v                                        v
+-----------------------------------------------------------------------------------+
|                         Cryptographic Signing Layer                               |
|                                                                                   |
|  - PKCS12 Production Keystores (homophones_keystore.jks, Release Keystores)       |
|  - Environment Variable Credential Injection (Air-Gapped from Version Control)     |
|  - R8 De-obfuscation Mapping Generation (mapping.txt)                             |
+-------------------------------------+---------------------------------------------+
                                      |
                                      v
+-----------------------------------------------------------------------------------+
|                        Google Play Release Progression                            |
|                                                                                   |
|  1. Internal App Sharing  --> 2. Closed Testing Track  --> 3. Staged Production   |
|     (Smoke & IAP Sandbox)        (20 Testers / 14 Days)       (10% -> 100% Roll)  |
|                                                                                   |
|  - Google Play App Signing (Split APK Generation per ABI / Density / Language)    |
|  - Android Vitals Monitoring (Crash Rate < 1.09%, ANR Rate < 0.47%)               |
|  - Firebase Crashlytics Real-Time Telemetry & De-obfuscated Stack Traces          |
+-----------------------------------------------------------------------------------+
```

## Core Tech Stack

- **Build Systems & Compilers**: Android Gradle Plugin (AGP 8.9.3, AGP 8.6.1), Gradle 8.11+ / 8.7, Kotlin Compiler (2.0.0, 2.1.20), Java Development Kit (JDK 17 LTS), Android NDK (28.1.13356709), Kotlin Symbol Processing (KSP 2.0.0-1.0.24), KAPT.
- **Android SDK & Runtime**: Android SDK Platform 35 (Android 15), Build Tools 35.0.1, AndroidX Jetpack Compose BOM (2025.05.01), Material 3 (1.3.2), Room Persistence Library (2.7.1), Android Core Splashscreen (1.0.1).
- **Packaging & Delivery**: Android App Bundle (AAB), Play Asset Delivery (`com.android.asset-pack` install-time delivery), ABI Filters (`arm64-v8a`), Legacy Native Library Packaging controls.
- **Security & Shrinking**: Java KeyStore (PKCS12 standard, RSA 2048/4096-bit asymmetric keys), ProGuard / R8 code shrinking and resource stripping, signature-protected custom permissions.
- **Store Compliance & Telemetry**: Google Play Console, Google User Messaging Platform (UMP SDK 3.2.0), Google Mobile Ads SDK (24.3.0), Google Play In-App Billing Client, Firebase BoM (33.14.0), Firebase Crashlytics (19.4.3), Google Play Android Vitals.

---

# PART 2: Release Engineering & SDLC Logic

## 1. Build & Compilation Strategy

### Target SDKs and API Level Compliance

Google Play requires target API level compliance to ensure applications operate under modern Android security and execution sandboxes. The build configurations across this repository target Android 15 (API level 35).

| Application | Project Type | Compile SDK | Target SDK | Min SDK | AGP Version | Language Runtime |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Branching Realms** | Native Android | `35` | `35` | `26` (Android 8.0) | `8.9.3` | Kotlin 2.0.0 / Java 17 |
| **Scratchboard Art Studio** | Godot Engine Port | `35` | `35` | `24` (Android 7.0) | `8.6.1` | Kotlin 2.1.20 / NDK 28 / Java 17 |
| **Homophones** | Production Keystore | `35` | `35` | `26` (Android 8.0) | `8.x` | JVM Bytecode / Java 17 |

- **Minimum SDK 26 (Branching Realms)**: Establishes baseline support for Java 8 language APIs natively, adaptive icons, and background execution limits without multidex complexity or backward compatibility overhead.
- **Minimum SDK 24 (Scratchboard Art Studio)**: Accommodates low-level Vulkan graphics drivers and OpenGL ES 3.0 requirements across older Android devices while supporting modern 64-bit ARM architectures.
- **Target SDK 35**: Complies with Google Play's policy requiring new apps and updates to target Android 14 (API level 34) or Android 15 (API level 35), enforcing strict runtime permissions, foreground service restrictions, and package visibility limits.

### Dependency Management & Compiler Conflict Resolution

Dependency graphs can cause compilation failures when mixing modern compiler plugins, annotation processors, and third-party SDKs. The native and engine build scripts resolve these issues via strict dependency governance.

#### Bill of Materials (BOM) Architecture
`BranchingRealms/app/build.gradle.kts` uses Gradle Version Catalogs (`gradle/libs.versions.toml`) combined with Google BOMs to prevent version drift:

```kotlin
dependencies {
    // Firebase BoM controls versions across Auth, Firestore, Analytics, Crashlytics, and Messaging
    implementation(platform(libs.firebase.bom)) // 33.14.0
    implementation(libs.firebase.auth.ktx)
    implementation(libs.firebase.firestore)
    implementation(libs.firebase.analytics.ktx)
    implementation(libs.firebase.crashlytics.ktx)
    implementation(libs.firebase.messaging)

    // Compose BoM standardizes UI library versions
    implementation(platform(libs.androidx.compose.bom)) // 2025.05.01
    implementation(libs.androidx.ui)
    implementation(libs.androidx.material3)
    implementation(libs.androidx.navigation.compose)
}
```

#### Strict Kotlin Version Enforcement
To prevent metadata mismatch crashes between KAPT (used by Dagger Hilt 2.49) and KSP (used by Room 2.7.1), the Gradle configuration enforces a unified Kotlin compiler version across all transitive dependencies:

```kotlin
configurations.all {
    resolutionStrategy.eachDependency {
        if (requested.group == "org.jetbrains.kotlin") {
            useVersion("2.0.0")
            because("Avoid kapt breaking due to Kotlin metadata version mismatch")
        }
    }
}
```

#### Godot NDK and Native Conflict Resolution
`scratchboard-art/android/build/build.gradle` compiles native C++ libraries alongside Java wrappers. To resolve collisions when third-party libraries ship duplicate C++ runtimes, the build configuration enforces a strict resolution rule:

```groovy
packagingOptions {
    pickFirst 'lib/x86/libc++_shared.so'
    pickFirst 'lib/x86_64/libc++_shared.so'
    pickFirst 'lib/armeabi-v7a/libc++_shared.so'
    pickFirst 'lib/arm64-v8a/libc++_shared.so'
}
```

The script cleans conflicting vendor native libraries during the `MergeNativeLibsTask` execution phase, removing redundant OpenXR vendor binaries (`libopenxr_loader.so`) before packaging.

#### Play Asset Delivery (PAD) Integration
Scratchboard Art Studio isolates large graphic assets from the main application binary through a dedicated Gradle asset pack module (`assetPackInstallTime/build.gradle`):

```groovy
plugins {
    id 'com.android.asset-pack'
}
assetPack {
    packName = "assetPackInstallTime"
    // Install-time asset delivery configuration
    deliveryType = "install-time"
}
```

This configuration packages the game assets directly into the Android App Bundle (AAB), downloading them automatically during initial store installation without requiring post-install HTTP downloads or violating Play Store size limits.

### Transition from Debug APKs to Production-Signed Android App Bundles (AAB)

The release pipeline enforces a separation between local development artifacts and store deployment packages:

```
[Local Development]
./gradlew assembleDebug 
       │
       ▼
Universal APK (android_debug.apk) ──> adb install (Local Test Devices / Emulators)

[Production Deployment]
./gradlew bundleRelease 
       │
       ▼
Android App Bundle (.aab) ──> Google Play Console Ingestion
                                      │
                                      ▼
                        Google Play App Signing (Store Key)
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            ▼                         ▼                         ▼
   Base Split APK             Config Split APKs          Asset Pack Split
(arm64-v8a native code)    (xxhdpi / en-US / etc.)   (assetPackInstallTime)
```

1. **Local Development (APKs)**: Running `gradlew assembleDebug` or `assembleStandardDebug` produces unoptimized, debug-signed, universal APKs (`android_debug.apk`). These contain all ABIs, resource densities, and unstripped symbols, facilitating fast deployment via ADB to local hardware.
2. **Production Compilation (AABs)**: Executing `gradlew bundleRelease` or exporting via Godot's export pipeline produces a single `.aab` file (`Scratchboard Art.aab`, `app-release.aab`). 
3. **Play Feature Delivery & Hardware-Targeted Splits**: Google Play ingests the AAB, verifies its cryptographic signature, and generates optimized split APKs tailored to each user's specific hardware:
   - **ABI Split**: Distributes only `arm64-v8a` native code to 64-bit devices, excluding unused `armeabi-v7a` or `x86_64` binaries.
   - **Screen Density Split**: Serves only the target device's drawable bucket (e.g., `xxhdpi`), stripping unneeded densities.
   - **Language Split**: Delivers strings matching the user's system locale.
   - **Install-Time Asset Packs**: Delivers the game assets packaged in `assetPackInstallTime` directly alongside the base APK during installation.

This compilation strategy reduces the user download footprint by 30% to 60% compared to monolithic APKs.

---

## 2. Cryptographic Signing & Security

### Keystore Management & Isolation

Android requires all installed packages to carry a valid cryptographic signature. The repository separates debug development certificates from production release keys.

```
+------------------------------------------------------------------------------------+
|                               Keystore Architecture                                |
|                                                                                    |
|  [Development Environments]               [Production Release Pipeline]            |
|  ~/.android/debug.keystore                C:\Dev\Android\Homophones\               |
|  - Insecure default password              homophones_keystore.jks                  |
|  - Auto-generated RSA certificate         - Air-gapped PKCS12 production keystore  |
|  - Binds only to debug builds             - SHA-256 certificate fingerprint        |
|                                           - Credentials injected via environment   |
|                                           - Applied to release bundles exclusively |
+------------------------------------------------------------------------------------+
```

- **Debug Signing**: Development builds utilize standard debug keystores automatically created by AGP, using known, public credentials (`android / androiddebugkey`). These keys never touch production release builds.
- **Production Keystores (`homophones_keystore.jks`)**: Release signing relies on dedicated PKCS12 key stores holding 2048-bit or 4096-bit RSA private keys with SHA-256 signatures. The `Homophones` project isolates its production keystore from open version control.
- **Credential Protection**: Build scripts avoid hardcoding passwords or file locations. Release signing configurations query environment variables or local, git-ignored properties files at build time:
  ```groovy
  signingConfigs {
      release {
          File keystoreFile = new File(getReleaseKeystoreFile())
          if (keystoreFile.isFile()) {
              storeFile keystoreFile
              storePassword getReleaseKeystorePassword()
              keyAlias getReleaseKeyAlias()
              keyPassword getReleaseKeystorePassword()
          }
      }
  }
  ```
- **Google Play App Signing**: The production pipeline uses Google Play App Signing. The local keystore signs the bundle as an **Upload Key**. Upon ingestion, Google Play validates the upload certificate, strips the upload signature, and signs the distributed split APKs with the master app signing key stored within Google's cloud key management infrastructure.

### ProGuard and R8 Optimization

Release builds activate R8 to shrink code, strip unused resources, and obfuscate identifiers.

#### Code and Resource Stripping Configuration
In `BranchingRealms/app/build.gradle.kts`:
```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
    }
}
```
In `scratchboard-art/android/build/gradle.properties`:
```properties
android.enableResourceOptimizations=true
```

#### R8 Execution Mechanics
1. **Dead Code Elimination (Tree Shaking)**: Traverses the entry points declared in `AndroidManifest.xml` (Activities, Services, Receivers) and prunes unreachable classes, interfaces, methods, and fields introduced by large dependencies like Firebase, Hilt, and Room.
2. **Resource Pruning**: Removes unreferenced resources (XML files, drawables, strings) from packaged binaries after code shrinking identifies unused identifiers.
3. **Identifier Obfuscation**: Renames remaining classes, methods, and member variables to short, non-descriptive names (`a`, `b`, `c`), reducing DEX file size and preventing decompilation.

#### Mapping File Pipeline for Telemetry
Because R8 obfuscates method names and line numbers, stack traces received from field crashes are unreadable without the corresponding mapping file. The build pipeline integrates the Firebase Crashlytics Gradle plugin (`com.google.firebase.crashlytics`):
- During `./gradlew bundleRelease`, R8 outputs an obfuscation translation map to `build/outputs/mapping/release/mapping.txt`.
- The Crashlytics plugin reads `mapping.txt` and uploads it to Firebase and Google Play Console.
- Incoming crash reports are automatically re-traced to original Kotlin source files and line numbers in real time.

---

## 3. Permissions & Manifest Compliance

### Manifest Auditing & Least-Privilege Scoping

Google Play developer policies strictly regulate the permissions apps can request. Unnecessary access to sensitive data (location, contacts, storage, SMS, camera) leads to store rejection. The manifests across both native and Godot projects enforce a least-privilege architecture.

#### Scratchboard Art Studio: Complete Zero-Trust Surface
In `scratchboard-art/export_presets.cfg`, every single dangerous Android permission is explicitly set to `false`. The application requests zero access to:
- Location (`access_fine_location=false`, `access_coarse_location=false`)
- External Storage (`read_external_storage=false`, `write_external_storage=false`, `manage_external_storage=false`)
- Camera & Microphone (`camera=false`, `record_audio=false`)
- Personal Data (`read_contacts=false`, `read_calendar=false`, `read_sms=false`)

The resulting merged manifest (`scratchboard-art/android/build/build/intermediates/merged_manifests/standardDebug/processStandardDebugManifest/AndroidManifest.xml`) permits only network communication required for Google Mobile Ads and Google Play Billing:
```xml
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
```

#### Branching Realms: Scoped Modern Permissions
In `BranchingRealms/app/src/main/AndroidManifest.xml` and its merged release manifest:
- **Notification Scoping (Android 13+ / API 33)**: Declares `android.permission.POST_NOTIFICATIONS` explicitly, requesting runtime consent before posting Firebase Cloud Messaging alerts.
- **Broadcast Isolation (Android 14+ / API 34)**: Declares a custom signature-protected permission to prevent unauthorized inter-app communication with internal receivers:
  ```xml
  <permission
      android:name="com.example.branchingrealms.RUNTIME_RECEIVER_NOT_EXPORTED_PERMISSION"
      android:protectionLevel="signature" />
  <uses-permission android:name="com.example.branchingrealms.RUNTIME_RECEIVER_NOT_EXPORTED_PERMISSION" />
  ```
- **Haptic Feedback**: Requests `android.permission.VIBRATE` to support tactile UI interactions without touching user privacy boundaries.
- **Package Visibility (`<queries>`)**: Adheres to Android 11+ package visibility rules by explicitly declaring intent queries for HTTPS browsing, Custom Tabs, and basic telephony, preventing broad package enumeration.

### Storage Sandboxing & Scoped Storage

Neither application requests legacy external storage permissions.
- In `scratchboard-art`:
  ```xml
  android:requestLegacyExternalStorage="false"
  android:allowBackup="false"
  android:hasFragileUserData="false"
  ```
- In `BranchingRealms`:
  Data persistence is restricted to internal sandboxed directories using Room and Jetpack DataStore. Backup behavior is controlled via XML declarations:
  ```xml
  android:fullBackupContent="@xml/backup_rules"
  android:dataExtractionRules="@xml/data_extraction_rules"
  ```
  This setup prevents sensitive authentication tokens, database encryption keys, or private user preferences from leaking into unencrypted cloud backups or device-to-device migration transfers.

### Component Encapsulation & Secure Activity Aliases

Exported activities represent exposed entry points that other applications can invoke via explicit or implicit intents.
- In `BranchingRealms`, entry points (`SplashActivity`, `MainActivity`, `ConsentActivity`) declare explicit `android:exported` values, satisfying Android 12+ security requirements. Internal services like `MyFirebaseMessagingService` set `android:exported="false"`.
- In `scratchboard-art`, the main engine activity (`.GodotApp`) sets `android:exported="false"`. Launch execution is routed through an explicit activity alias:
  ```xml
  <activity
      android:name=".GodotApp"
      android:exported="false"
      android:screenOrientation="landscape" />

  <activity-alias
      android:name=".GodotAppLauncher"
      android:targetActivity=".GodotApp"
      android:exported="true">
      <intent-filter>
          <action android:name="android.intent.action.MAIN" />
          <category android:name="android.intent.category.DEFAULT" />
          <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
  </activity-alias>
  ```
  This pattern encapsulates internal engine states while exposing only a controlled launcher interface to the Android OS.

### Privacy Compliance: User Messaging Platform (UMP) & Consent Checks

To comply with Google Play's Families Policy and European Economic Area (EEA) ePrivacy / GDPR mandates, apps displaying ads must capture user consent via a Google-certified Consent Management Platform (CMP).

`BranchingRealms` implements this directly in `com.example.branchingrealms.auth.UMPManager`:
- Initializes Google's User Messaging Platform SDK (`user-messaging-platform:3.2.0`).
- Gathers user consent before loading or initializing Google Mobile Ads (`com.google.android.gms.ads.APPLICATION_ID`).
- Restricts test debug geography settings strictly to development builds:
  ```kotlin
  if (BuildConfig.DEBUG) {
      val debugSettings = ConsentDebugSettings.Builder(context)
          .setDebugGeography(ConsentDebugSettings.DebugGeography.DEBUG_GEOGRAPHY_EEA)
          .addTestDeviceHashedId("3487F2022C8042A6F7B8A8C672F4F770")
          .build()
      builder.setConsentDebugSettings(debugSettings)
  }
  ```
- Gates telemetry collection: `CrashlyticsHelper.isCrashReportingEnabled(context)` verifies user consent flags from local preferences before transmitting operational crash reports to Firebase.

---

## 4. Testing Tracks & Validation

Shipping release bundles to production requires a structured progression across Google Play Console distribution tracks to isolate defects and protect user experience.

```
                    Google Play Release Progression
                    
+--------------------------------------------------------------------------+
| Track 1: Internal App Sharing & Testing                                  |
| - Fast turnaround builds for development teams                           |
| - Target: Smoke tests, Play Asset Delivery validation, IAP sandbox checks|
| - Audience: Up to 100 internal accounts                                  |
+------------------------------------+-------------------------------------+
                                     |
                                     v
+--------------------------------------------------------------------------+
| Track 2: Closed Testing Track (Google Play Compliance Gate)              |
| - Mandatory requirement for personal developer accounts created post-2023|
| - Target: Maintain 20+ opt-in testers continuously for 14+ days          |
| - Feedback: Android Vitals evaluation, device fragmentation, crash checks|
+------------------------------------+-------------------------------------+
                                     |
                                     v
+--------------------------------------------------------------------------+
| Track 3: Staged Production Rollout                                       |
| - Phase 1: 10% rollout -> Monitor real-time Vitals & Crashlytics         |
| - Phase 2: 25% rollout -> Verify ANR / Crash thresholds remain clear     |
| - Phase 3: 50% rollout -> Expand hardware / OS demographic distribution |
| - Phase 4: 100% rollout -> Full production release                       |
+--------------------------------------------------------------------------+
```

### Track Progression Strategy

#### 1. Internal App Sharing & Internal Testing Track
- **Purpose**: Rapid distribution for immediate smoke testing.
- **Validation Scope**:
  - Verifies that the signed `.aab` installs correctly via the Play Store client engine.
  - Validates that `assetPackInstallTime` delivers all required binary game assets on launch without missing asset exceptions.
  - Tests Google Play Billing responses using licensed test accounts in sandbox mode.
  - Tests Google Sign-In and Firebase Authentication against production SHA-256 certificate fingerprints.

#### 2. Closed Testing Track (20 Opt-in Testers Policy)
Google requires all personal developer accounts created after November 13, 2023, to run a closed test with a minimum of 20 opt-in testers who remain actively opted in for at least 14 consecutive days before applying for production access.
- **Operational Management**:
  - Creates dedicated email list groups in Google Play Console.
  - Distributes the closed testing web and Android opt-in URLs.
  - Enforces daily active testing across 20+ diverse hardware profiles (differing screen sizes, RAM tiers, Android OS versions 8.0 through 15).
  - Collects non-public bug reports, UI clipping issues, and performance logs via Play Console feedback forms.
  - Monitors the Closed Testing dashboard daily to ensure the opt-in count remains consistently at or above the 20-tester minimum threshold for the full 14-day duration.

#### 3. Production Staged Rollout
Once Google Play approves the application for production, updates are deployed using phased percentage releases:
1. **10% Staged Rollout**: Deploys to a random 10% sample of production users. The release halts immediately if unexpected crashes emerge.
2. **25% Staged Rollout**: Increases distribution after 24 hours of stable telemetry.
3. **50% Staged Rollout**: Broadens geographic and device distribution after 48 hours.
4. **100% Full Rollout**: Completes the deployment once all operational gates pass.

If critical defects appear during any stage, the rollout is halted directly from the Play Console, preventing further distribution while engineers prepare an expedited patch release.

---

### Quality Gates, ANRs, and Crash Rate Monitoring

Production health is governed by continuous telemetry streaming into Google Play Android Vitals and Firebase Crashlytics.

#### Android Vitals Enforcement Thresholds
Google Play algorithms penalize or remove apps that exceed standard "bad behavior" thresholds:

| Metric | Google Play Bad Behavior Threshold | Production Target | Action on Violation |
| :--- | :--- | :--- | :--- |
| **User-Perceived Crash Rate** | `> 1.09%` | `< 0.20%` | Immediate Rollout Halt; R8 de-obfuscation root-cause analysis |
| **User-Perceived ANR Rate** | `> 0.47%` | `< 0.10%` | Main thread audit; profiling background coroutines & Room I/O |
| **Crash-Free Users (Crashlytics)** | `< 99.0%` | `> 99.8%` | Block staged rollout progression to next percentage tier |

#### Telemetry Mechanics & Crash Resolution
1. **Application Not Responding (ANR) Prevention**:
   - In `BranchingRealms`, heavy disk I/O and database operations (Room queries, DataStore updates) are offloaded from the main UI thread onto `Dispatchers.IO` via Kotlin Coroutines.
   - In `scratchboard-art`, resource loading is executed asynchronously to ensure the rendering loop maintains consistent framerates and avoids triggering the 5-second Android input dispatch timeout.
2. **Firebase Crashlytics Breadcrumb Logging**:
   - `BranchingRealms` logs key state transitions into `CrashlyticsHelper` (e.g., registration attempts, authentication events, UMP consent state).
   - When an unhandled exception or non-fatal error occurs, the recorded breadcrumbs accompany the stack trace, exposing the exact user interaction sequence leading up to the failure.
3. **Symbol Table De-obfuscation**:
   - Automated mapping uploads during bundle compilation ensure that production stack traces appear fully de-obfuscated in the developer console, identifying the precise Kotlin class and source line number responsible for the error.
