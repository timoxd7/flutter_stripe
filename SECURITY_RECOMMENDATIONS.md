# Security Recommendations for flutter_stripe

This document provides actionable security recommendations for developers using and maintaining the flutter_stripe library.

## For Library Maintainers

### High Priority Fixes

#### 1. Android: Add URL Validation in openAuthenticatedWebView

**File:** `packages/stripe_android/android/src/main/kotlin/com/flutter/stripe/StripeAndroidPlugin.kt`

**Current Code (Line ~301):**
```kotlin
"openAuthenticatedWebView" -> {
    stripeSdk.openAuthenticatedWebView(
        id = call.requiredArgument("id"),
        url = call.requiredArgument<String>("url")
    )
}
```

**Recommended Fix:**
```kotlin
"openAuthenticatedWebView" -> {
    val url = call.requiredArgument<String>("url")
    
    // Validate URL is HTTPS
    if (!url.startsWith("https://")) {
        result.error("INVALID_URL", "URL must use HTTPS protocol", null)
        return
    }
    
    // Validate URL is from stripe.com domain
    try {
        val uri = Uri.parse(url)
        if (!uri.host?.endsWith("stripe.com") == true) {
            result.error("INVALID_URL", "URL must be from stripe.com domain", null)
            return
        }
    } catch (e: Exception) {
        result.error("INVALID_URL", "Malformed URL", null)
        return
    }
    
    stripeSdk.openAuthenticatedWebView(
        id = call.requiredArgument("id"),
        url = url
    )
}
```

#### 2. Android: Replace .orEmpty() with Proper Validation

**Files:** Multiple files in `stripe_android` package

**Current Pattern:**
```kotlin
val merchantCountryCode = getString("merchantCountryCode").orEmpty()
```

**Recommended Pattern:**
```kotlin
val merchantCountryCode = getString("merchantCountryCode")
    ?: throw IllegalArgumentException("merchantCountryCode is required for this operation")
```

#### 3. iOS: Replace Force Unwraps with Safe Unwrapping

**File:** `packages/stripe_ios/ios/Classes/stripe_ios/Sources/StripeSdkImpl.swift`

**Current Pattern (Line ~87):**
```swift
let publishableKey = call.arguments["publishableKey"] as! String
```

**Recommended Pattern:**
```swift
guard let publishableKey = call.arguments["publishableKey"] as? String else {
    result(FlutterError(
        code: "INVALID_ARGUMENT",
        message: "publishableKey is required and must be a string",
        details: nil
    ))
    return
}
```

#### 4. iOS: Wrap Debug Print Statements

**Files:** Multiple Swift files in `stripe_ios` package

**Current Pattern:**
```swift
print("URL Scheme: \(urlScheme)")
```

**Recommended Pattern:**
```swift
#if DEBUG
print("URL Scheme: \(urlScheme)")
#endif
```

#### 5. iOS: Move Sensitive Config to Keychain

**File:** `packages/stripe_ios/ios/Classes/stripe_ios/Sources/StripeSdkImpl.swift`

**Current Code:**
```swift
UserDefaults.standard.set(liveMode, forKey: "stripe_userKeyLiveMode")
```

**Recommended Code:**
```swift
// Use Keychain for sensitive configuration
import Security

func saveToKeychain(value: Bool, forKey key: String) {
    let data = Data([value ? 1 : 0])
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecValueData as String: data
    ]
    
    SecItemDelete(query as CFDictionary)
    SecItemAdd(query as CFDictionary, nil)
}

saveToKeychain(value: liveMode, forKey: "stripe_userKeyLiveMode")
```

### Medium Priority Improvements

#### 6. Android: Add Network Security Configuration

**File:** Create `packages/stripe_android/android/src/main/res/xml/network_security_config.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Disable cleartext traffic globally -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    
    <!-- Stripe domain configuration -->
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">stripe.com</domain>
        <domain includeSubdomains="true">api.stripe.com</domain>
        <!-- Add certificate pins here in future -->
    </domain-config>
</network-security-config>
```

**Update:** `packages/stripe_android/android/src/main/AndroidManifest.xml`
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application
        android:networkSecurityConfig="@xml/network_security_config">
    </application>
</manifest>
```

#### 7. Android: Enhance ProGuard Rules

**File:** `packages/stripe_android/android/proguard-rules.txt`

**Add:**
```proguard
# Keep Stripe classes (already present, but enhance)
-keep class com.stripe.** { *; }
-keepclassmembers class com.stripe.** { *; }

# Prevent obfuscation issues with Stripe SDK
-dontwarn com.stripe.android.pushProvisioning.**
-dontwarn kotlinx.parcelize.**

# Remove debug logging in release builds
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
    public static *** i(...);
}

# Keep line numbers for stack traces
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# Preserve annotations for runtime inspection
-keepattributes *Annotation*

# Keep generic signature for proper type inference
-keepattributes Signature

# Optimize and obfuscate
-optimizationpasses 5
-dontusemixedcaseclassnames
-dontskipnonpubliclibraryclasses
-dontpreverify
```

#### 8. iOS: Add App Transport Security Configuration

**File:** Developers should add to their `Info.plist`

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>
    <key>NSExceptionDomains</key>
    <dict>
        <key>stripe.com</key>
        <dict>
            <key>NSIncludesSubdomains</key>
            <true/>
            <key>NSExceptionRequiresForwardSecrecy</key>
            <true/>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.2</string>
        </dict>
    </dict>
</dict>
```

#### 9. Web: Harden Script Injection

**File:** `packages/stripe_js/lib/src/loader/stripe_loader.dart`

**Current Code (Line 10-35):**
```dart
void _injectSrcScript(String src, String windowVar) {
  final script = web.HTMLScriptElement();
  script.type = 'module';
  script.crossOrigin = 'anonymous';
  script.text = '''
    window.ff_trigger_$windowVar = async (callback) => {
      callback(await import("$src"));
    };
  ''';
  // ...
}
```

**Recommended Enhancement:**
```dart
void _injectSrcScript(String src, String windowVar) {
  // Validate inputs to prevent injection
  assert(src.startsWith('https://'), 'Script source must use HTTPS');
  assert(RegExp(r'^[a-zA-Z0-9_]+$').hasMatch(windowVar), 
         'Window variable name must be alphanumeric');
  
  final script = web.HTMLScriptElement();
  script.type = 'module';
  script.crossOrigin = 'anonymous';
  
  // Use const values to prevent injection
  const allowedSources = ['https://js.stripe.com/v3/'];
  if (!allowedSources.contains(src)) {
    throw ArgumentError('Unauthorized script source: $src');
  }
  
  script.text = '''
    window.ff_trigger_$windowVar = async (callback) => {
      callback(await import("$src"));
    };
  ''';
  // ...
}
```

### Low Priority Enhancements

#### 10. Add Dependency Security Scanning to CI/CD

**File:** `.github/workflows/security.yaml` (create new)

```yaml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 * * 0'  # Weekly scan

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Dart dependency audit
        run: |
          flutter pub get
          flutter pub audit
      
      - name: Run Android dependency check
        run: |
          cd packages/stripe_android/android
          ./gradlew dependencyCheckAnalyze
      
      - name: Upload results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: build/reports/dependency-check-report.sarif
```

#### 11. Document Security Best Practices

**File:** `SECURITY.md` (create new)

```markdown
# Security Policy

## Reporting Security Vulnerabilities

**DO NOT** open public issues for security vulnerabilities.

Instead, please report security issues privately through:
- GitHub Security Advisory: [Create Advisory](https://github.com/flutter-stripe/flutter_stripe/security/advisories/new)
- Email: [maintainer email]

We will respond within 48 hours and work with you to address the issue.

## Security Best Practices

### For Developers Using This Library

1. **Never hardcode API keys**
   - Use environment variables for publishable keys
   - Never commit secret keys to version control

2. **PCI Compliance**
   - Do NOT use `dangerouslyGetFullCardDetails()` in production
   - Use Stripe Elements or PaymentSheet for card input
   - Never log or store card numbers

3. **API Key Security**
   - Use test keys (pk_test_*) in development
   - Rotate keys if compromised
   - Restrict key permissions in Stripe Dashboard

4. **Server-Side Validation**
   - Always validate payments server-side
   - Never trust client-side payment confirmation
   - Use webhooks for payment status updates

### For Library Maintainers

1. **Dependency Updates**
   - Keep Stripe SDKs updated monthly
   - Monitor security advisories
   - Run `flutter pub audit` regularly

2. **Code Review**
   - Require security review for payment-related changes
   - Test with security tools (SAST, DAST)
   - Validate input in all public APIs

3. **Release Process**
   - Sign releases
   - Publish security changelogs
   - Notify users of critical updates
```

## For Developers Using This Library

### Quick Security Checklist

- [ ] Store publishable keys in environment variables, not in code
- [ ] Use test keys (pk_test_*) in development
- [ ] Use live keys (pk_live_*) only in production
- [ ] Never use `dangerouslyGetFullCardDetails()` unless you understand PCI implications
- [ ] Validate all payments server-side with your backend
- [ ] Use webhooks for payment status updates
- [ ] Implement rate limiting for payment attempts
- [ ] Enable fraud detection in Stripe Dashboard
- [ ] Use 3D Secure for European customers (automatic with this library)
- [ ] Test error handling for declined cards
- [ ] Log payment attempts (but never card details)
- [ ] Implement CSP headers if using web
- [ ] Keep flutter_stripe updated to latest version

### Secure Configuration Examples

#### Flutter/Dart
```dart
// ✅ GOOD: Load from environment
const publishableKey = String.fromEnvironment('STRIPE_PUBLISHABLE_KEY');
Stripe.publishableKey = publishableKey;

// ❌ BAD: Hardcoded key
Stripe.publishableKey = 'pk_live_...'; // Never do this!
```

#### Android
```kotlin
// Use BuildConfig for keys (not recommended for production)
// Better: Fetch from secure backend endpoint

// In build.gradle
android {
    defaultConfig {
        buildConfigField "String", "STRIPE_KEY", "\"${System.getenv('STRIPE_KEY')}\""
    }
}
```

#### iOS
```swift
// Use Info.plist with environment-specific configurations
// Or fetch from secure backend endpoint

guard let path = Bundle.main.path(forResource: "Config", ofType: "plist"),
      let config = NSDictionary(contentsOfFile: path),
      let stripeKey = config["StripePublishableKey"] as? String else {
    fatalError("Missing Stripe configuration")
}
```

### Server-Side Integration

**Always validate payments on your backend:**

```javascript
// Node.js example
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);

app.post('/create-payment-intent', async (req, res) => {
  // Validate request
  // Calculate amount server-side (never trust client)
  const amount = calculateAmount(req.body.items);
  
  const paymentIntent = await stripe.paymentIntents.create({
    amount: amount,
    currency: 'usd',
    automatic_payment_methods: { enabled: true },
  });
  
  res.json({ clientSecret: paymentIntent.client_secret });
});

// Verify payment with webhooks
app.post('/webhook', async (req, res) => {
  const sig = req.headers['stripe-signature'];
  let event;
  
  try {
    event = stripe.webhooks.constructEvent(
      req.body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }
  
  if (event.type === 'payment_intent.succeeded') {
    // Fulfill order
    const paymentIntent = event.data.object;
    await fulfillOrder(paymentIntent);
  }
  
  res.json({ received: true });
});
```

## Testing Security

### Security Test Checklist

- [ ] Test with invalid publishable keys
- [ ] Test with expired payment methods
- [ ] Test with insufficient funds
- [ ] Test with invalid card numbers
- [ ] Test with network interruptions
- [ ] Test with malformed responses
- [ ] Test rate limiting
- [ ] Test concurrent payment attempts
- [ ] Verify error messages don't leak sensitive data
- [ ] Verify logs don't contain card numbers

### Penetration Testing

Consider running these security tests:
1. **OWASP Mobile Top 10** - Test for common mobile vulnerabilities
2. **Man-in-the-Middle** - Verify HTTPS and certificate validation
3. **Code Injection** - Test input validation
4. **Data Storage** - Verify no card data is stored
5. **Memory Dump Analysis** - Verify sensitive data is cleared

## Incident Response

If you discover a security issue:

1. **Immediate Actions:**
   - Rotate compromised API keys in Stripe Dashboard
   - Review payment logs for suspicious activity
   - Disable affected payment methods if necessary

2. **Investigation:**
   - Determine scope of exposure
   - Identify affected customers
   - Document timeline of events

3. **Remediation:**
   - Apply security patches
   - Update library version
   - Test fix thoroughly

4. **Notification:**
   - Notify affected users if required by law
   - Report to relevant authorities
   - Update security documentation

## Additional Resources

- [Stripe Security Best Practices](https://stripe.com/docs/security/best-practices)
- [PCI DSS Compliance Guide](https://stripe.com/docs/security/guide)
- [OWASP Mobile Security](https://owasp.org/www-project-mobile-security/)
- [Flutter Security Best Practices](https://flutter.dev/security)

---

**Last Updated:** January 15, 2026  
**Maintained By:** flutter_stripe security team
