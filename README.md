# Meta (Facebook) App Events — Complete Flutter Integration Guide

> **Production-tested guide** extracted from a live Flutter app.
> Follow each section **in order** and you'll have Meta analytics tracking working from scratch.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Meta Developer Console Setup](#2-meta-developer-console-setup)
3. [Flutter Dependencies](#3-flutter-dependencies)
4. [Android Native Configuration](#4-android-native-configuration)
5. [iOS Native Configuration](#5-ios-native-configuration)
6. [Dart Code — Architecture & Implementation](#6-dart-code--architecture--implementation)
7. [Initialization & Lifecycle](#7-initialization--lifecycle)
8. [Tracked Events Reference](#8-tracked-events-reference)
9. [iOS App Tracking Transparency (ATT)](#9-ios-app-tracking-transparency-att)
10. [Privacy & Data Sanitization](#10-privacy--data-sanitization)
11. [ProGuard / R8 Rules (Android)](#11-proguard--r8-rules-android)
12. [Testing & Debugging](#12-testing--debugging)
13. [Production Checklist](#13-production-checklist)
14. [Troubleshooting](#14-troubleshooting)

---

## 1. Prerequisites

Before you start, make sure you have:

- [ ] A **Meta (Facebook) Developer Account** → [developers.facebook.com](https://developers.facebook.com)
- [ ] A **Meta App** created in the Meta Developer Console
- [ ] Your app's **App ID** and **Client Token** from Meta
- [ ] Flutter SDK ≥ 3.9 (Dart 3+)
- [ ] A working Flutter project targeting **Android** and/or **iOS**

---

## 2. Meta Developer Console Setup

### Step 1: Create a Meta App

1. Go to [Meta for Developers](https://developers.facebook.com/apps/)
2. Click **"Create App"**
3. Select app type: **"Business"** (for analytics/ads) or **"Consumer"**
4. Fill in:
   - **App Name**: Your app name (e.g., `My App`)
   - **App Contact Email**: Your email
   - **Business Account**: Select or create one
5. Click **"Create App"**

### Step 2: Get Your App ID & Client Token

1. Go to **Settings → Basic**:
   - Copy the **App ID** (e.g., `xxxxxxxxxxxxxxxxx`)
   - Copy the **App Secret** (for server-to-server only, NOT used in the mobile app)
2. Go to **Settings → Advanced**:
   - Find **Client Token** (e.g., `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`)
   - If it doesn't exist, click **"Generate"**

> [!IMPORTANT]
> The **Client Token** is different from the **App Secret**. The Client Token is safe to embed in mobile apps. The App Secret must NEVER be in client code.

### Step 3: Add Your Platform

#### For Android:
1. Go to **Settings → Basic → Add Platform → Android**
2. Fill in:
   - **Package Name**: `com.example.myapp` (your `applicationId`)
   - **Class Name**: `com.example.myapp.MainActivity`
   - **Key Hashes**: Generate with:
     ```bash
     # Debug key hash
     keytool -exportcert -alias androiddebugkey -keystore ~/.android/debug.keystore | openssl sha1 -binary | openssl base64
     
     # Release key hash (use YOUR keystore path and alias)
     keytool -exportcert -alias YOUR_KEY_ALIAS -keystore /path/to/your.keystore | openssl sha1 -binary | openssl base64
     ```
3. **Save Changes**

#### For iOS:
1. Go to **Settings → Basic → Add Platform → iOS**
2. Fill in:
   - **Bundle ID**: `com.example.myapp` (your `PRODUCT_BUNDLE_IDENTIFIER`)
3. **Save Changes**

### Step 4: Enable App Events

1. Go to **App Dashboard → Events Manager**
2. Make sure **App Events** is enabled for your app
3. Under **Settings → Basic**, ensure:
   - ✅ "Log App Events" is ON

### Step 5: Set App Mode to Live

> [!WARNING]
> Your Meta app starts in **Development Mode**. In this mode, only app admins/developers/testers see events. Switch to **Live Mode** before production release.

1. Go to **App Dashboard** → Toggle **"App Mode"** from `Development` to `Live`
2. You'll need to complete:
   - Privacy Policy URL
   - Data Deletion callback or instructions URL
   - Accepted Meta's terms

---

## 3. Flutter Dependencies

Add these to your `pubspec.yaml`:

```yaml
dependencies:
  # Meta (Facebook) App Events SDK
  facebook_app_events: 0.30.5
  
  # iOS App Tracking Transparency (required for iOS 14.5+)
  app_tracking_transparency: ^2.0.7
```

Then run:
```bash
flutter pub get
```

> [!NOTE]
> `facebook_app_events` wraps the native Meta SDK for both Android and iOS. The `app_tracking_transparency` package is needed for the iOS ATT prompt (required by Apple since iOS 14.5).

---

## 4. Android Native Configuration

### 4.1 `android/app/src/main/res/values/strings.xml`

Create or update this file with your Meta credentials:

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- Replace with your actual Meta App ID and Client Token from Meta Developer portal -->
    <string name="facebook_app_id">YOUR_META_APP_ID</string>
    <string name="facebook_client_token">YOUR_META_CLIENT_TOKEN</string>
</resources>
```

**Example:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="facebook_app_id">xxxxxxxxxxxxxxxxx</string>
    <string name="facebook_client_token">xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx</string>
</resources>
```

### 4.2 `android/app/src/main/AndroidManifest.xml`

Add these inside the `<application>` tag:

```xml
<application
    android:label="Your App Name"
    android:name="${applicationName}"
    android:icon="@mipmap/ic_launcher">

    <!-- ====== META SDK CONFIGURATION ====== -->
    
    <!-- Meta App ID (required) -->
    <meta-data
        android:name="com.facebook.sdk.ApplicationId"
        android:value="@string/facebook_app_id" />
    
    <!-- Meta Client Token (required since SDK v13+) -->
    <meta-data
        android:name="com.facebook.sdk.ClientToken"
        android:value="@string/facebook_client_token" />
    
    <!-- Auto-log app install, app launch, and in-app purchase events -->
    <meta-data
        android:name="com.facebook.sdk.AutoLogAppEventsEnabled"
        android:value="true" />
    
    <!-- Collect Advertiser ID for ad attribution -->
    <meta-data
        android:name="com.facebook.sdk.AdvertiserIDCollectionEnabled"
        android:value="true" />
    
    <!-- ====== END META SDK CONFIGURATION ====== -->
    
    <!-- ... rest of your activities, services, etc. -->
</application>
```

### 4.3 Permissions

Add the **AD_ID** permission (required on Android 13+ for ad attribution):

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Required for Meta App Events ad attribution on Android 13+ -->
    <uses-permission android:name="com.google.android.gms.permission.AD_ID" />
    
    <!-- Also make sure you have INTERNET permission -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    
    <!-- ... -->
</manifest>
```

> [!IMPORTANT]
> Without the `AD_ID` permission on Android 13+, Meta cannot attribute ad conversions. Google Play also requires you to declare this in the Data Safety section.

---

## 5. iOS Native Configuration

### 5.1 `ios/Runner/Info.plist`

Add the following keys inside the top-level `<dict>`:

```xml
<!-- ====== META SDK CONFIGURATION ====== -->

<!-- Facebook URL Scheme (required for SDK initialization) -->
<key>CFBundleURLTypes</key>
<array>
    <!-- ... your other URL schemes ... -->
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>fb YOUR_META_APP_ID</string>
            <!-- Example: fbxxxxxxxxxxxxxxxxx (no space — "fb" prefix + your App ID) -->
        </array>
    </dict>
</array>

<!-- Meta App ID -->
<key>FacebookAppID</key>
<string>YOUR_META_APP_ID</string>

<!-- Meta Client Token -->
<key>FacebookClientToken</key>
<string>YOUR_META_CLIENT_TOKEN</string>

<!-- Display name shown in Facebook dialogs -->
<key>FacebookDisplayName</key>
<string>Your App Name</string>

<!-- Auto-log app events (install, launch, purchases) -->
<key>FacebookAutoLogAppEventsEnabled</key>
<true/>

<!-- Collect Advertiser ID for ad attribution -->
<key>FacebookAdvertiserIDCollectionEnabled</key>
<true/>

<!-- ====== APP TRACKING TRANSPARENCY (ATT) ====== -->

<!-- Required by Apple since iOS 14.5 — this is the text shown in the ATT dialog -->
<key>NSUserTrackingUsageDescription</key>
<string>This permission helps us provide personalized offers and discounts tailored to your interests.</string>

<!-- Allow Meta SDK to query its companion apps -->
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>fbapi</string>
    <string>fb-messenger-share-api</string>
</array>

<!-- ====== END META SDK CONFIGURATION ====== -->
```

**Example (complete values):**
```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>fbxxxxxxxxxxxxxxxxx</string>
        </array>
    </dict>
</array>

<key>FacebookAppID</key>
<string>xxxxxxxxxxxxxxxxx</string>

<key>FacebookClientToken</key>
<string>xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx</string>

<key>FacebookDisplayName</key>
<string>My App</string>

<key>FacebookAutoLogAppEventsEnabled</key>
<true/>

<key>FacebookAdvertiserIDCollectionEnabled</key>
<true/>

<key>NSUserTrackingUsageDescription</key>
<string>This permission helps us provide personalized offers and discounts tailored to your interests.</string>

<key>LSApplicationQueriesSchemes</key>
<array>
    <string>fbapi</string>
    <string>fb-messenger-share-api</string>
</array>
```

> [!TIP]
> The `NSUserTrackingUsageDescription` text is what users see in the iOS ATT permission dialog. Make it clear and user-friendly — Apple rejects vague descriptions. Write it in your app's language.

### 5.2 `ios/Podfile`

Make sure your minimum iOS deployment target is **≥ 13.0** (Meta SDK requirement). Recommended: **15.0**.

```ruby
platform :ios, '15.0'
```

### 5.3 Install Pods

```bash
cd ios
pod install --repo-update
cd ..
```

---

## 6. Dart Code — Architecture & Implementation

The integration uses a clean architecture pattern with an abstract contract and a concrete implementation. This allows swapping the analytics provider without touching business logic.

### 6.1 Abstract Contract — `analytics_tracker.dart`

> File: `lib/core/analytics/analytics_tracker.dart`

```dart
/// ISO 4217 currency used for every monetary analytics value in the app.
const String kAnalyticsDefaultCurrency = 'EGP';

/// Content types sent with analytics events. Keep this list small and stable
/// because campaign reports group by these values.
abstract final class AnalyticsContentType {
  static const String merchant = 'merchant';
  static const String product = 'product';
}

/// Contract for product analytics.
///
/// Implementations must never throw: a failing analytics call must not break
/// login, cart or checkout flows. Callers must also never pass personal or
/// financial data (phone, address, notes, card data, exact location).
abstract class AnalyticsTracker {
  /// Prepares the underlying SDK. Safe to call without awaiting.
  Future<void> init();

  /// A new account was created successfully on the server.
  Future<void> logCompletedRegistration({required String method});

  /// A search was submitted (not on every keystroke).
  Future<void> logSearch({required String query});

  Future<void> logMerchantView({required String merchantId});

  Future<void> logProductView({
    required String productId,
    required num price,
    String currency = kAnalyticsDefaultCurrency,
  });

  /// [value] is the total value of the quantity that was just added, not the
  /// unit price.
  Future<void> logAddToCart({
    required String productId,
    required num quantity,
    required num value,
    String currency = kAnalyticsDefaultCurrency,
  });

  Future<void> logInitiateCheckout({
    required num total,
    required int itemCount,
    String currency = kAnalyticsDefaultCurrency,
  });

  /// Must be called only after the server confirmed the order was created.
  Future<void> logPurchase({
    required num total,
    required int itemCount,
    required String orderType,
    String? orderId,
    String currency = kAnalyticsDefaultCurrency,
  });

  /// Sets the unique user identifier for analytics tracking.
  Future<void> setUserId(String userId);

  /// Requests iOS App Tracking Transparency (ATT) authorization dialog.
  Future<void> requestTrackingAuthorization();

  /// Clears user data and user ID from the analytics SDK (e.g. on logout).
  Future<void> clearUserData();
}
```

### 6.2 Meta Implementation — `meta_analytics_service.dart`

> File: `lib/core/analytics/meta_analytics_service.dart`

```dart
import 'package:app_tracking_transparency/app_tracking_transparency.dart';
import 'package:facebook_app_events/facebook_app_events.dart';
import 'package:flutter/foundation.dart';
import 'package:sentry_flutter/sentry_flutter.dart';

import 'analytics_tracker.dart';

/// Meta (Facebook) App Events implementation of [AnalyticsTracker].
///
/// The service stays disabled (every call is a silent no-op) when:
///  * the platform has no Meta SDK (web, desktop), or
///  * no Meta App ID is configured natively yet.
/// This lets the app ship safely before the Meta keys are added.
class MetaAnalyticsService implements AnalyticsTracker {
  MetaAnalyticsService({FacebookAppEvents? events})
    : _events = events ?? FacebookAppEvents();

  static const int _maxSearchLength = 50;

  final FacebookAppEvents _events;
  bool _enabled = false;

  @visibleForTesting
  bool get isEnabled => _enabled;

  /// Returns a privacy-safe search string, or `null` when it must not be sent
  /// (empty, or looks like an email / phone number typed by the user).
  @visibleForTesting
  static String? sanitizeSearchQuery(String raw) {
    final normalized = raw.trim().replaceAll(RegExp(r'\s+'), ' ');
    if (normalized.isEmpty) return null;
    if (normalized.contains('@')) return null;
    final digitCount = RegExp(r'\d').allMatches(normalized).length;
    if (digitCount >= 7) return null;
    return normalized.length > _maxSearchLength
        ? normalized.substring(0, _maxSearchLength)
        : normalized;
  }

  bool get _isPlatformSupported =>
      !kIsWeb &&
      (defaultTargetPlatform == TargetPlatform.android ||
          defaultTargetPlatform == TargetPlatform.iOS);

  @override
  Future<void> init() async {
    if (!_isPlatformSupported) return;
    try {
      final appId = await _events.getApplicationId();
      _enabled = appId != null &&
          appId.trim().isNotEmpty &&
          !appId.contains('[') &&
          appId != '0';
      if (!_enabled) {
        debugPrint('MetaAnalytics: no valid App ID configured, tracking disabled');
        return;
      }

      if (kDebugMode) {
        await _events.setDebugLoggingEnabled(true);
        debugPrint('MetaAnalytics: initialized with App ID $appId (debug logging enabled)');
      }

      await _events.setAdvertiserIdCollectionEnabled(true);
      await _events.setAutoLogAppEventsEnabled(true);
      await _events.activateApp();
    } catch (error, stackTrace) {
      _enabled = false;
      _reportFailure('init', error, stackTrace);
    }
  }

  @override
  Future<void> requestTrackingAuthorization() async {
    if (kIsWeb || defaultTargetPlatform != TargetPlatform.iOS) return;
    try {
      final status = await AppTrackingTransparency.trackingAuthorizationStatus;
      if (status == TrackingStatus.notDetermined) {
        final newStatus =
            await AppTrackingTransparency.requestTrackingAuthorization();
        if (kDebugMode) {
          debugPrint('MetaAnalytics: ATT requested, status = $newStatus');
        }
      }
      final currentStatus =
          await AppTrackingTransparency.trackingAuthorizationStatus;
      final authorized = currentStatus == TrackingStatus.authorized;
      await _events.setAdvertiserIdCollectionEnabled(authorized);
    } catch (error, stackTrace) {
      _reportFailure('request_tracking_authorization', error, stackTrace);
    }
  }

  @override
  Future<void> setUserId(String userId) {
    if (userId.isEmpty) return Future.value();
    return _run('set_user_id', () => _events.setUserID(userId));
  }

  @override
  Future<void> clearUserData() {
    return _run('clear_user_data', () async {
      await _events.clearUserID();
      await _events.clearUserData();
    });
  }

  @override
  Future<void> logCompletedRegistration({required String method}) {
    return _run(
      'complete_registration',
      () => _events.logCompletedRegistration(registrationMethod: method),
    );
  }

  @override
  Future<void> logSearch({required String query}) {
    final sanitized = sanitizeSearchQuery(query);
    if (sanitized == null) return Future.value();
    return _run('search', () => _events.logSearched(searchString: sanitized));
  }

  @override
  Future<void> logMerchantView({required String merchantId}) {
    if (merchantId.isEmpty) return Future.value();
    return _run(
      'view_merchant',
      () => _events.logViewContent(
        id: merchantId,
        type: AnalyticsContentType.merchant,
      ),
    );
  }

  @override
  Future<void> logProductView({
    required String productId,
    required num price,
    String currency = kAnalyticsDefaultCurrency,
  }) {
    if (productId.isEmpty) return Future.value();
    return _run(
      'view_product',
      () => _events.logViewContent(
        id: productId,
        type: AnalyticsContentType.product,
        currency: currency,
        price: price.toDouble(),
      ),
    );
  }

  @override
  Future<void> logAddToCart({
    required String productId,
    required num quantity,
    required num value,
    String currency = kAnalyticsDefaultCurrency,
  }) {
    if (productId.isEmpty) return Future.value();
    return _run(
      'add_to_cart',
      () => _events.logAddToCart(
        content: {'quantity': quantity},
        id: productId,
        type: AnalyticsContentType.product,
        currency: currency,
        price: value.toDouble(),
      ),
    );
  }

  @override
  Future<void> logInitiateCheckout({
    required num total,
    required int itemCount,
    String currency = kAnalyticsDefaultCurrency,
  }) {
    return _run(
      'initiate_checkout',
      () => _events.logInitiatedCheckout(
        totalPrice: total.toDouble(),
        currency: currency,
        contentType: AnalyticsContentType.product,
        numItems: itemCount,
      ),
    );
  }

  @override
  Future<void> logPurchase({
    required num total,
    required int itemCount,
    required String orderType,
    String? orderId,
    String currency = kAnalyticsDefaultCurrency,
  }) {
    return _run(
      'purchase',
      () => _events.logPurchase(
        amount: total.toDouble(),
        currency: currency,
        parameters: {
          FacebookAppEvents.paramNameNumItems: itemCount,
          FacebookAppEvents.paramNameContentType: AnalyticsContentType.product,
          'order_type': orderType,
          if (orderId != null && orderId.isNotEmpty)
            FacebookAppEvents.paramNameOrderId: orderId,
        },
      ),
    );
  }

  /// Runs [action] only when enabled and swallows every failure so analytics
  /// can never break a business flow.
  Future<void> _run(String event, Future<void> Function() action) async {
    if (!_enabled) return;
    try {
      await action();
      if (kDebugMode) {
        debugPrint('MetaAnalytics: logged event "$event"');
        await _events.flush();
      }
    } catch (error, stackTrace) {
      _reportFailure(event, error, stackTrace);
    }
  }

  void _reportFailure(String event, Object error, StackTrace stackTrace) {
    debugPrint('MetaAnalytics: "$event" failed: ${error.runtimeType}');
    // Only the error type/stack are reported, never event parameters.
    Sentry.captureException(error, stackTrace: stackTrace);
  }
}
```

### 6.3 Dependency Injection — `get_it` Registration

> File: `lib/core/dependancy_injection/dependancy_injection.dart`

```dart
import 'package:get_it/get_it.dart';
import '../analytics/analytics_tracker.dart';
import '../analytics/meta_analytics_service.dart';

final getIt = GetIt.instance;

Future<void> setupGetIt() async {
  // Register as a lazy singleton — one instance for the entire app lifetime
  getIt.registerLazySingleton<AnalyticsTracker>(() => MetaAnalyticsService());
  
  // ... other registrations
}
```

> [!NOTE]
> Registered as `AnalyticsTracker` (the abstract type), NOT `MetaAnalyticsService`. This means all consumers depend on the contract, making it trivial to swap to a different provider (e.g., Firebase Analytics, Mixpanel) without any code changes.

---

## 7. Initialization & Lifecycle

### 7.1 App Startup (`main.dart`)

```dart
import 'dart:async';
import 'core/analytics/analytics_tracker.dart';
import 'core/dependancy_injection/dependancy_injection.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // 1. Register dependencies (including MetaAnalyticsService)
  setupGetIt();
  
  // 2. Initialize analytics SDK (fire-and-forget — don't block app startup)
  unawaited(getIt<AnalyticsTracker>().init());
  
  // 3. If user is already logged in, set their user ID
  final loggedInUserId = /* read from cache */;
  if (loggedInUserId != null && loggedInUserId.isNotEmpty) {
    unawaited(getIt<AnalyticsTracker>().setUserId(loggedInUserId));
  }
  
  runApp(const MyApp());
  
  // 4. Request ATT permission AFTER the first frame is rendered (iOS only)
  //    Delayed to avoid showing the dialog before the user sees the app UI
  WidgetsBinding.instance.addPostFrameCallback((_) {
    Future.delayed(const Duration(milliseconds: 1000), () {
      getIt<AnalyticsTracker>().requestTrackingAuthorization();
    });
  });
}
```

> [!IMPORTANT]
> **Key decisions:**
> - `init()` is called with `unawaited()` — analytics should NEVER block app startup
> - ATT is requested 1 second after the first frame — Apple will reject your app if you show the dialog before any UI is rendered
> - `setUserId()` is called early so all subsequent events are tied to the user

### 7.2 On Login

```dart
// In your Auth Cubit / login success handler:
_analytics.setUserId(user.id);
```

### 7.3 On Registration

```dart
// After server confirms new account creation:
_analytics.logCompletedRegistration(method: 'phone');
```

### 7.4 On Logout

```dart
// In your cache-clearing / logout logic:
if (getIt.isRegistered<AnalyticsTracker>()) {
  getIt<AnalyticsTracker>().clearUserData();
}
```

---

## 8. Tracked Events Reference

| Meta Event | Method | When to Fire | Parameters |
|---|---|---|---|
| `fb_mobile_complete_registration` | `logCompletedRegistration()` | After server confirms new account | `method` (e.g., `'phone'`) |
| `fb_mobile_search` | `logSearch()` | On search submit (not keystroke) | `query` (sanitized) |
| `fb_mobile_content_view` | `logMerchantView()` | User opens a merchant/store page | `merchantId` |
| `fb_mobile_content_view` | `logProductView()` | User opens a product detail page | `productId`, `price`, `currency` |
| `fb_mobile_add_to_cart` | `logAddToCart()` | User adds item to cart | `productId`, `quantity`, `value`, `currency` |
| `fb_mobile_initiated_checkout` | `logInitiateCheckout()` | User starts checkout flow | `total`, `itemCount`, `currency` |
| `fb_mobile_purchase` | `logPurchase()` | Server confirms order created | `total`, `itemCount`, `orderType`, `orderId`, `currency` |

> [!TIP]
> These are **standard Meta events**. Using standard events (instead of custom ones) means Meta can automatically optimize your ad campaigns for conversions, build lookalike audiences, and provide out-of-the-box reporting.

---

## 9. iOS App Tracking Transparency (ATT)

Since iOS 14.5, Apple **requires** asking user permission before tracking them across apps/websites.

### What the Code Does

1. **Checks current status** — if already determined (authorized/denied), skips the dialog
2. **Shows the ATT dialog** — only if status is `notDetermined`
3. **Adjusts Advertiser ID collection** — enables only if the user authorized tracking

### ATT Dialog Flow

```
App launches → UI renders → 1 second delay → ATT dialog appears
                                                    │
                                        ┌───────────┴───────────┐
                                        │                       │
                                   "Allow"                  "Ask App
                                        │                  Not to Track"
                                        │                       │
                              Enable Advertiser ID     Disable Advertiser ID
                              collection (full         collection (limited
                              ad attribution)          attribution)
```

### Important Notes

- ATT is **iOS only** — the code does nothing on Android
- The dialog text comes from `NSUserTrackingUsageDescription` in `Info.plist`
- Users can change their decision later in **Settings → Privacy → Tracking**
- Even if denied, basic analytics (event counts) still work — only ad attribution is limited

---

## 10. Privacy & Data Sanitization

The implementation includes built-in privacy protection:

### Search Query Sanitization

Before any search query is sent to Meta, it goes through `sanitizeSearchQuery()`:

```
Input                      → Output
"pizza delivery"           → "pizza delivery"      ✅ Sent
"   lots   of   spaces  " → "lots of spaces"       ✅ Normalized & sent
""                         → null                   ❌ Blocked (empty)
"user@email.com"           → null                   ❌ Blocked (contains @)
"01012345678"              → null                   ❌ Blocked (≥7 digits = phone number)
"very long query...(51+)"  → "very long query..."   ✅ Truncated to 50 chars
```

### Rules Enforced

- ❌ **Never send**: emails, phone numbers, personal data
- ❌ **Never send**: empty strings or whitespace-only strings
- ✅ **Truncate** search queries to 50 characters max
- ✅ **Normalize** whitespace (collapse multiple spaces to one)
- ✅ Errors report **only error type + stack trace** to Sentry — **never event parameters**

---

## 11. ProGuard / R8 Rules (Android)

When `isMinifyEnabled = true` in your `build.gradle.kts`, R8/ProGuard can strip Meta SDK classes. Add these rules:

> File: `android/app/proguard-rules.pro`

```pro
# Facebook / Meta SDK ProGuard Rules
-keep class com.facebook.** { *; }
-keepclassmembers class com.facebook.** { *; }
-keepattributes Signature, *Annotation*, InnerClasses, EnclosingMethod
```

Make sure your `build.gradle.kts` references the ProGuard file:

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = false
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"     // ← This line
        )
    }
}
```

---

## 12. Testing & Debugging

### 12.1 Debug Logging

In **debug mode**, the service automatically:
- Enables Meta's debug logging (`setDebugLoggingEnabled(true)`)
- Prints every event to the console: `MetaAnalytics: logged event "purchase"`
- Calls `flush()` after each event to send it immediately (production batches events)

### 12.2 Meta Events Manager

1. Go to [Meta Events Manager](https://www.facebook.com/events_manager2/)
2. Select your app
3. Click **"Test Events"** tab
4. Use your **real test device** (simulator may not work)
5. Events should appear within seconds in debug mode

### 12.3 Android — Verify Events via ADB

```bash
# Enable verbose logging for Facebook SDK
adb shell setprop log.tag.FBSDKLog VERBOSE

# Watch the logs
adb logcat | grep -i "facebook\|FBSDKLog\|MetaAnalytics"
```

### 12.4 iOS — Verify via Xcode Console

Filter Xcode console output for:
- `MetaAnalytics:`
- `FBSDKLog`

### 12.5 Graceful Degradation

The service is designed to **never crash the app**:
- If Meta App ID is missing → tracking silently disables (`_enabled = false`)
- If the platform is web/desktop → all calls are no-ops
- If any event fails → error is caught, logged to Sentry, app continues normally
- All methods check for empty IDs and return early

---

## 13. Production Checklist

### Meta Developer Console
- [ ] Meta App is in **Live Mode** (not Development)
- [ ] Privacy Policy URL is set
- [ ] Data Deletion callback/instructions URL is set
- [ ] Android platform added with correct **package name** and **key hashes** (both debug & release)
- [ ] iOS platform added with correct **Bundle ID**
- [ ] App Events product is enabled

### Android
- [ ] `strings.xml` has `facebook_app_id` and `facebook_client_token`
- [ ] `AndroidManifest.xml` has all 4 `<meta-data>` entries (ApplicationId, ClientToken, AutoLogAppEvents, AdvertiserIDCollection)
- [ ] `AndroidManifest.xml` has `AD_ID` permission
- [ ] `proguard-rules.pro` has Meta SDK keep rules
- [ ] `build.gradle.kts` references `proguard-rules.pro`
- [ ] Google Play Data Safety form declares: App Events, Advertising ID

### iOS
- [ ] `Info.plist` has `FacebookAppID`, `FacebookClientToken`, `FacebookDisplayName`
- [ ] `Info.plist` has `FacebookAutoLogAppEventsEnabled` = true
- [ ] `Info.plist` has `FacebookAdvertiserIDCollectionEnabled` = true
- [ ] `Info.plist` has `CFBundleURLSchemes` with `fb{YOUR_APP_ID}` (e.g., `fb1547544027139873`)
- [ ] `Info.plist` has `NSUserTrackingUsageDescription` (ATT dialog text)
- [ ] `Info.plist` has `LSApplicationQueriesSchemes` with `fbapi` and `fb-messenger-share-api`
- [ ] `Podfile` minimum deployment target ≥ 13.0
- [ ] `pod install --repo-update` has been run after adding dependencies
- [ ] App Store Connect: App Privacy section is filled out

### Dart Code
- [ ] `pubspec.yaml` has `facebook_app_events` and `app_tracking_transparency`
- [ ] `AnalyticsTracker` abstract class is defined
- [ ] `MetaAnalyticsService` implements `AnalyticsTracker`
- [ ] Registered in `get_it` as `AnalyticsTracker` (singleton)
- [ ] `init()` called in `main()` with `unawaited()`
- [ ] `requestTrackingAuthorization()` called after first frame (with delay)
- [ ] `setUserId()` called on login and cached session restore
- [ ] `clearUserData()` called on logout
- [ ] Business events (search, view, add-to-cart, checkout, purchase) are wired in cubits

### Privacy
- [ ] Search queries are sanitized (no emails, no phone numbers)
- [ ] Error reports to Sentry contain no event parameters
- [ ] No personal data (phone, address, card info) is passed to analytics

---

## 14. Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `MetaAnalytics: no valid App ID configured` | Missing/wrong App ID in native config | Check `strings.xml` (Android) or `Info.plist` (iOS) |
| Events don't appear in Events Manager | App is in Development Mode | Switch to Live Mode in Meta console |
| Events don't appear in Test Events | Using simulator | Use a real device for testing |
| `ClassNotFoundException: com.facebook.` | ProGuard stripping Meta classes | Add ProGuard rules (Section 11) |
| ATT dialog not showing | Already determined, or called before UI | Check `TrackingStatus`, ensure delay after first frame |
| `AD_ID` permission warning in Play Console | Missing permission declaration | Add `com.google.android.gms.permission.AD_ID` to manifest |
| iOS build fails with `Facebook` pod error | CocoaPods cache stale | Run `cd ios && pod deintegrate && pod install --repo-update` |
| Events fire in debug but not release | ProGuard or different App ID | Verify release key hash is added in Meta console |

---

## File Structure Summary

```
project/
├── pubspec.yaml                          # facebook_app_events + app_tracking_transparency
├── android/
│   └── app/
│       ├── build.gradle.kts              # ProGuard reference, minification
│       ├── proguard-rules.pro            # Meta SDK keep rules
│       └── src/main/
│           ├── AndroidManifest.xml        # Meta meta-data + AD_ID permission
│           └── res/values/
│               └── strings.xml           # facebook_app_id + facebook_client_token
├── ios/
│   ├── Podfile                           # platform :ios, '15.0'
│   └── Runner/
│       └── Info.plist                    # FacebookAppID, ClientToken, ATT, URL schemes
└── lib/
    └── core/
        ├── analytics/
        │   ├── analytics_tracker.dart    # Abstract contract
        │   └── meta_analytics_service.dart # Meta implementation
        └── dependancy_injection/
            └── dependancy_injection.dart  # get_it registration
```

---

## License

This guide is released under the **MIT License**. Use it, share it, improve it.

---

> **Made with ❤️ — tested in production on both Android and iOS.**
