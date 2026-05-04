# Pitfalls Research

**Domain:** React Native TurboModule library — system-provided modal document scanner (VisionKit + ML Kit)
**Researched:** 2026-05-04
**Confidence:** HIGH (platform APIs, build tooling, and community patterns verified against official sources and issue trackers)

---

## Critical Pitfalls

### Pitfall 1: Wrong `create-react-native-library` Template Silently Produces Old-Arch Shim

**What goes wrong:**
The `turbo-module-mixed` template (vs `turbo-module`) generates both a TurboModule and a legacy bridge shim. The shim compiles cleanly, so `xcodebuild` and `assembleDebug` pass — but the shipped library exposes an `RCTBridgeModule` conformance that enables old-arch consumers you explicitly don't want to support. Worse, choosing the wrong template also injects `oldArchEnabled=true` gradle flags that silently disable codegen validation on Android.

**Why it happens:**
The CRNL interactive prompt lists `turbo-module` and `turbo-module-mixed` adjacently. `turbo-module-mixed` sounds safer ("supports more consumers"), and the description mentions "backward compat" — which sounds like good practice until you realise it doubles your maintenance surface and contradicts the new-arch-only learning goal.

**How to avoid:**
In the scaffold step (Chunk 1), always select `turbo-module` (no backward compat). Verify immediately after scaffolding: `package.json` must NOT contain `"codegen": { "ios": { "type": "all" } }` — `all` means both TM and bridge. It must say `"type": "turbo"` only. If `ios/RNSystemScannerModule.h` contains `RCT_EXTERN_MODULE`, the wrong template was chosen and the scaffold must be redone.

**Warning signs:**
- `RCT_EXTERN_MODULE` appears in generated `.h` file
- `turbo-module-mixed` appears in the scaffold confirmation text
- `package.json` has `codegenConfig.ios.type: "all"` instead of `"turbo"`

**Phase to address:** Chunk 1 (Scaffold). Verify immediately after `npx create-react-native-library@latest` completes, before any code is written.

---

### Pitfall 2: Swift Auto-Generated Header Name Mismatch Causes Opaque Compile Failure

**What goes wrong:**
The Obj-C++ bridge file imports the auto-generated Swift header as `#import "react_native_system_scanner-Swift.h"`. Xcode generates this header name from the podspec `s.name` field by replacing all non-alphanumeric characters (hyphens, dots) with underscores. If the podspec name differs from what you expect, or if you type the import manually before running `pod install`, the compile fails with: `'react_native_system_scanner-Swift.h' file not found` — a confusing error that doesn't point to the root cause.

**Why it happens:**
`create-react-native-library` doesn't scaffold Swift files at all. The Swift header import must be added manually post-scaffold. Developers often guess the header name from the package name (`react-native-system-scanner` → `react_native_system_scanner`) without knowing that Xcode uses the podspec `s.name` — which may be set differently. The `-Swift.h` suffix is mandatory and must appear verbatim.

**How to avoid:**
After `pod install`, check `DerivedData/<app>/Build/Intermediates.noindex/Pods.build/Debug-iphonesimulator/<PodName>.build/Swift\ Compatibility\ Header/` to find the actual generated header name. Do not write the import until after pod install has been run once. Add a comment in the `.mm` file documenting the derivation rule: `// Header name = s.name with hyphens→underscores, plus "-Swift.h"`.

**Warning signs:**
- `file not found` on the `-Swift.h` import line during `xcodebuild`
- Build succeeds only after `pod deintegrate && pod install` (stale DerivedData had old name)
- Changing `s.name` in the podspec without updating the import breaks builds silently

**Phase to address:** Chunk 2 (iOS native). Address as the very first step of Swift wiring, before writing any logic.

---

### Pitfall 3: `VNDocumentCameraViewController` Delegate Retain Cycle Leaks the TurboModule

**What goes wrong:**
`RNSystemScannerImpl` is set as the delegate of `VNDocumentCameraViewController`. VisionKit's `delegate` property is declared `weak` in the framework headers — which is correct. However, if `RNSystemScannerImpl` holds a strong reference to the `VNDocumentCameraViewController` instance while the VC also holds the delegate strongly through an intermediate closure or completion handler, a cycle forms. The typical failure mode: after the first scan, the Swift object is not deallocated, the `resolve`/`reject` blocks it captured are retained forever, and the next `scan()` call stacks a second presenter on top of a VC that was never dismissed properly.

**Why it happens:**
In a TurboModule context, `RNSystemScannerImpl` is typically a class-level singleton or long-lived object. If it captures `self` strongly in a closure that is stored inside the VC (e.g., a completion handler passed as a property), ARC cannot break the cycle. The typical mistake: storing the `VNDocumentCameraViewController` instance as a strong `var` on `RNSystemScannerImpl` without nilifying it in all delegate callback paths.

**How to avoid:**
- In `RNSystemScannerImpl`, declare the VC reference as `weak`: `weak var documentCamera: VNDocumentCameraViewController?`
- In all three delegate methods (`didFinishWith`, `didCancel`, `didFailWithError`): dismiss the VC, call `resolve`/`reject`, then immediately nil out `self.pendingResolve` and `self.pendingReject`
- Use `[weak self]` in any closure that references `self` from inside the VC or its delegate methods
- After every scan in the example app, check Instruments → Leaks to verify zero allocations survive the modal dismissal

**Warning signs:**
- Successive `scan()` calls stack modals (VC was never released)
- Instruments → Allocations shows `RNSystemScannerImpl` instance count climbing with each scan
- The second `scan()` call resolves immediately with the first scan's stale result

**Phase to address:** Chunk 2 (iOS native). Must be verified with Instruments on a real device, not just by code review.

---

### Pitfall 4: Presenting `VNDocumentCameraViewController` When No Safe Presenter Exists

**What goes wrong:**
`scan()` is called during cold launch from a push notification (before the RN view hierarchy is mounted), or from a background notification handler, or during a scene transition. The attempt to find `UIApplication.shared.keyWindow?.rootViewController` (deprecated iOS 13+) or even the `UIWindowScene`-based equivalent returns `nil` or returns a `UIViewController` that is currently presenting another VC. The `present(_:animated:)` call fails with a runtime warning:

```
Warning: Attempt to present <VNDocumentCameraViewController> on <ViewController>
whose view is not in the window hierarchy
```

The promise never resolves. On iOS 13+, `UIApplication.shared.keyWindow` is deprecated, and calling it on an extension or notification target can return `nil` even when a window exists.

**Why it happens:**
RN modules execute JS logic as soon as the JS bridge initialises — before the native view hierarchy is ready. Notification delivery can trigger JS code (via PushNotificationIOS or local notification handlers) before `viewDidAppear` has fired on the root VC.

**How to avoid:**
```swift
// In RNSystemScannerImpl.swift
private func topMostViewController() -> UIViewController? {
    guard let windowScene = UIApplication.shared.connectedScenes
        .filter({ $0.activationState == .foregroundActive })
        .compactMap({ $0 as? UIWindowScene })
        .first,
        let window = windowScene.windows.first(where: { $0.isKeyWindow })
    else { return nil }

    var top = window.rootViewController
    while let presented = top?.presentedViewController {
        top = presented
    }
    return top
}
```
Guard on `nil` from `topMostViewController()` and resolve `{ kind: 'error', code: 'unavailable', message: "No active view controller found — call scan() after the app is in the foreground" }` rather than crashing.

**Warning signs:**
- Xcode console emits `Warning: Attempt to present... whose view is not in the window hierarchy`
- `scan()` called from a notification handler immediately after cold launch hangs indefinitely
- `rootViewController` is non-nil but has `presentedViewController != nil` (another sheet is open)

**Phase to address:** Chunk 2 (iOS native). Add the guard in the initial implementation; verify in the example app by testing from a notification cold launch.

---

### Pitfall 5: Android `pendingPromise` Leaked When Activity Is Destroyed Mid-Scan

**What goes wrong:**
`SystemScannerModule` stores the JS `Promise` in `var pendingPromise: Promise?`. If the host Activity is destroyed while the ML Kit scanner Activity is active (phone call, system kill, RAM pressure), `onActivityResult` is never called. `pendingPromise` stays non-null but is never resolved or rejected. The JS side hangs forever (no timeout). On Activity recreation, the same TurboModule instance may be created fresh — the old promise reference is now dangling.

**Why it happens:**
`ActivityEventListener.onActivityResult` is only called if the hosting Activity survives. Android's process death under memory pressure or a phone call terminating the foreground stack means the callback never fires. RN does not automatically reject dangling promises on process death.

**How to avoid:**
Implement the "pending result" rescue pattern:
1. Before `startIntentSenderForResult`, serialize enough state to recover: store `scanId` and a timestamp in a module-level field (not in the Activity saved state — the TurboModule outlives config changes but not process death)
2. In `onHostResume()` (implement `LifecycleEventListener` in addition to `ActivityEventListener`), check if `pendingPromise != null` and the last scan intent was issued more than N seconds ago — if so, reject with `'unavailable'`
3. Always clear `pendingPromise = null` at the top of `onActivityResult` (before the result branch) so double-fire cannot resolve twice

```kotlin
override fun onActivityResult(activity: Activity, requestCode: Int, resultCode: Int, data: Intent?) {
    if (requestCode != REQUEST_CODE) return
    val promise = pendingPromise ?: return  // already handled or never set
    pendingPromise = null                   // clear FIRST, before any async work
    // ... handle result
}
```

**Warning signs:**
- `scan()` never resolves after pressing the home button during an active scan session
- LogCat shows `ActivityManager: Process killed` during scanner activity
- Calling `scan()` a second time after a failed first scan causes "already resolving" errors from RN

**Phase to address:** Chunk 3 (Android native). Add `LifecycleEventListener` alongside `ActivityEventListener` from the start; write a Jest test that validates the timeout-reject path via a mock.

---

### Pitfall 6: `content://` URI Copy Deferred to a Non-`onActivityResult` Coroutine Scope

**What goes wrong:**
The content:// URIs returned by `GmsDocumentScanningResult` are scoped to the ML Kit scanner's ContentProvider and become invalid once the scanner Activity has fully finished. If `FileCopyHelper.copyResultAsync` is called with a coroutine that suspends (e.g., `launch(Dispatchers.IO) { ... }`) and the scanner process's ContentProvider has already torn down by the time the coroutine resumes, `contentResolver.openInputStream(uri)` throws `FileNotFoundException` with a misleading message like `"Bad file descriptor"`.

**Why it happens:**
The intuitive fix for "ContentResolver on main thread" (StrictMode violation) is to move the copy to `Dispatchers.IO`. But `launch { }` is non-blocking — it returns immediately and the coroutine runs later, possibly after the scanner process has released the content URI grant.

**How to avoid:**
Perform the copy **synchronously within `onActivityResult`** using `runBlocking { withContext(Dispatchers.IO) { ... } }`. Yes, this blocks the main thread briefly. The scanner modal has already dismissed; the user is waiting for the promise to resolve. For typical scans (1–10 pages at ~1MB each), the I/O completes in <500ms — acceptable. For very large scans, bound the copy to a coroutine tied to the module's lifecycle and ensure the content URI read starts before returning from `onActivityResult`:

```kotlin
override fun onActivityResult(activity: Activity, requestCode: Int, resultCode: Int, data: Intent?) {
    if (requestCode != REQUEST_CODE || resultCode != Activity.RESULT_OK) return
    val promise = pendingPromise ?: return
    pendingPromise = null

    val result = GmsDocumentScanningResult.fromActivityResultIntent(data) ?: run {
        promise.resolve(mapOf("kind" to "error", "code" to "unknown", "message" to "No result data"))
        return
    }

    // Blocking read — intentional; content:// URIs expire immediately after this returns
    runBlocking {
        withContext(Dispatchers.IO) {
            FileCopyHelper.copyResult(reactApplicationContext, result, scanId, promise)
        }
    }
}
```

**Warning signs:**
- `FileNotFoundException` or `"Bad file descriptor"` in the copy coroutine on high-page-count scans
- Works reliably on fast devices but fails intermittently on low-end Android (slower scanner teardown)
- StrictMode disk I/O violation in the debug build (expected and acceptable in this pattern — suppress for this call site with a custom policy if needed)

**Phase to address:** Chunk 3 (Android native). This is the single highest-severity runtime correctness issue on Android. Test with a 10-page scan on a slow emulator to confirm.

---

### Pitfall 7: `NSPrivacyInfo.xcprivacy` Missing — App Store Rejection Since May 2024

**What goes wrong:**
Apple has required `PrivacyInfo.xcprivacy` in all third-party SDKs that use "required reason APIs" since May 1, 2024. A library that writes files to `NSCachesDirectory`, accesses file timestamps (via `FileManager.attributesOfItem(atPath:)` to check for pre-existing scans), or queries disk space (`URLResourceValues.volumeAvailableCapacityForImportantUsageKey`) must declare the reason in this manifest. Apps using the library without the manifest get an App Store Connect rejection at submission time with email code `ITMS-91053`.

**Why it happens:**
The manifest requirement is relatively new (enforced mid-2024). Most library templates and tutorials predate it. `create-react-native-library` does not scaffold `PrivacyInfo.xcprivacy` for TurboModules. The library author ships without it; the consuming app gets rejected.

**How to avoid:**
Add `ios/PrivacyInfo.xcprivacy` as a plist file and declare it in the podspec:

```ruby
s.resource_bundles = {
  "react-native-system-scanner-privacy" => ["ios/PrivacyInfo.xcprivacy"]
}
```

Minimum required entries for this library:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <!-- FileManager.attributesOfItem, file URL stat() calls -->
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>C617.1</string>  <!-- "Inside your app or third-party SDK, managing files" -->
            </array>
        </dict>
        <!-- NSCachesDirectory queries (disk space available) -->
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryDiskSpace</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>85F4.1</string>  <!-- "Checking available space to write user-requested content" -->
            </array>
        </dict>
    </array>
    <key>NSPrivacyCollectedDataTypes</key>
    <array/>
    <key>NSPrivacyTracking</key>
    <false/>
</dict>
</plist>
```

Note: Camera access itself is declared via `NSCameraUsageDescription` in `Info.plist` (the config plugin handles this), not in the privacy manifest. The manifest covers required-reason APIs only.

**Warning signs:**
- App Store Connect email with code `ITMS-91053` mentioning `NSPrivacyAccessedAPICategoryFileTimestamp` or `NSPrivacyAccessedAPICategoryDiskSpace`
- Xcode Privacy Report (Product → Archive → Distribute → Privacy Manifest) lists the library as missing a manifest
- `pod install` completes without warning even when `PrivacyInfo.xcprivacy` is absent (no pod-level enforcement)

**Phase to address:** Chunk 2 (iOS native). Create the manifest alongside the Swift implementation; add the podspec `resource_bundles` entry in the same commit.

---

### Pitfall 8: ML Kit Model Unavailability Mapped as `'unsupported'` Instead of `'model_unavailable'`

**What goes wrong:**
`GmsDocumentScanning.getClient(options).getStartScanIntent(activity)` can fail even on a fully-supported device with current Play Services if the ML Kit model hasn't been downloaded yet and the device is offline. `MlKitException.UNAVAILABLE` (code 14) is thrown. If the module catches all `getStartScanIntent` failures as `'unsupported'`, the consumer shows "Your device doesn't support document scanning" — which is wrong and alarming — instead of "Check your internet connection and try again."

**Why it happens:**
The distinction between `MlKitException.UNAVAILABLE` (model not yet downloaded / service temporarily down) and a genuine GMS version incompatibility is subtle. The initial `isSupported()` check can pass (GMS version is OK) but the model download may not have completed yet (on-demand is the default).

**How to avoid:**
In `addOnFailureListener`, inspect the exception type:

```kotlin
.addOnFailureListener { e ->
    pendingPromise = null
    val (code, message) = when {
        e is MlKitException && e.errorCode == MlKitException.UNAVAILABLE ->
            "model_unavailable" to "ML Kit model not yet available. Check internet connection and try again."
        e is MlKitException && e.errorCode == MlKitException.NOT_ENOUGH_SPACE ->
            "unavailable" to "Insufficient device storage for ML Kit model."
        else ->
            "unavailable" to (e.message ?: "Scanner failed to launch")
    }
    promise.resolve(mapOf("kind" to "error", "code" to code, "message" to message))
}
```

Also wire up `ModuleInstall.getClient(context).areModulesAvailable(documentScanner)` as a preflight check inside `isSupported()` to distinguish "GMS absent" from "model not yet downloaded".

**Warning signs:**
- `isSupported()` returns `true` but `scan()` returns `{ kind: 'error', code: 'unsupported' }` on first run with no network
- LogCat: `MlKitException: UNAVAILABLE: The service is not available`
- Consistent on-demand model failure in CI emulator without network access to Google's model CDN

**Phase to address:** Chunk 3 (Android native). Handle in `addOnFailureListener`; add a test case in the example app's "no-GMS" scenario that also exercises the offline path.

---

### Pitfall 9: `TurboModuleRegistry.getEnforcing` Crash When Module Name in Spec Doesn't Match Registration

**What goes wrong:**
`TurboModuleRegistry.getEnforcing<Spec>('NativeSystemScanner')` crashes at startup with `Invariant Violation: TurboModuleRegistry.getEnforcing(...): 'NativeSystemScanner' could not be found. Verify that a module by this name is registered in the native binary.` This crash is immediate and fatal — the app won't start.

**Why it happens:**
Three names must align exactly (case-sensitive):
1. The string passed to `getEnforcing` in `NativeSystemScanner.ts`
2. The `name` field returned by `getName()` in the Kotlin module (`SystemScannerModule.kt`)
3. The `moduleName` in the iOS Obj-C++ `RCT_EXPORT_MODULE()` call

If any of these three drift — from a rename, a typo, or a template substitution error — the runtime cannot wire the JS call to the native module.

**How to avoid:**
Define the module name as a constant and use it in all three locations:
- `NativeSystemScanner.ts`: `TurboModuleRegistry.getEnforcing<Spec>('NativeSystemScanner')`
- `SystemScannerModule.kt`: `override fun getName() = NAME` where `const val NAME = "NativeSystemScanner"`
- `RNSystemScannerModule.mm`: `RCT_EXPORT_MODULE(NativeSystemScanner)`

The codegen-derived Kotlin `NativeSystemScannerSpec` class name is also derived from the spec file name (`NativeSystemScanner.ts`) — the `Native` prefix is mandatory for codegen discovery.

**Warning signs:**
- App crashes immediately on launch with `Invariant Violation: TurboModuleRegistry.getEnforcing`
- Android: `logcat` shows `"Module NativeSystemScanner could not be found"` at module registry time
- iOS: The error fires before any UI appears; Metro bundler console shows the JS stack trace

**Phase to address:** Chunk 1 (Scaffold + Types). Write the three names in sync from day one; add a Jest test that imports `NativeSystemScanner` and verifies the export is non-null (catches JS-side regressions).

---

## Moderate Pitfalls

### Pitfall 10: `react-native-builder-bob` ESM-Only Output Breaks Jest in Consumer Apps

**What goes wrong:**
`react-native-builder-bob` 0.40+ defaults to ESM-only output (no CJS fallback). Consumers running Jest with the default `react-native` Jest preset get `SyntaxError: Cannot use import statement in a module` when importing from this library, because Jest does not support ESM natively and the `main` field no longer points to a CJS file.

**Why it happens:**
The ESM-only change was intentional for Metro ≥ 0.82 consumers (which support `package.json` exports). But Jest uses Node's module system, which in Node < 20.19 cannot synchronously `require()` ESM. Most consumer Jest configs use `transformIgnorePatterns` that exclude `node_modules` — this library ends up untransformed and ESM.

**How to avoid:**
In the library's `package.json`, ensure `transformIgnorePatterns` guidance is documented in README:

```js
// jest.config.js in consumer app — they must add this
transformIgnorePatterns: [
  'node_modules/(?!(react-native-system-scanner)/)'
]
```

For the library itself: Jest tests for pure TS logic (result shape, error mapping) don't import native modules, so no Jest ESM issue exists in the library's own test suite. The problem is consumer-side only — document it in README under a "Testing" section.

Do NOT add a CJS output just for Jest compat — that reintroduces the dual package hazard. The documented `transformIgnorePatterns` fix is the correct answer.

**Warning signs:**
- Consumer's CI fails with `SyntaxError: Cannot use import statement` in their Jest suite
- The error references a path inside `node_modules/react-native-system-scanner/lib/`
- Works in the app itself (Metro handles ESM) but fails only in Jest

**Phase to address:** Chunk 1 (Scaffold + Types). Add the Jest config note to README in Chunk 6 (CI + Docs).

---

### Pitfall 11: pnpm Workspace `shamefully-hoist` Omission Causes Silent Metro Resolution Failures

**What goes wrong:**
The `example/` app uses Metro, which requires hoisted `node_modules` to resolve peer dependencies (React, React Native) from the library root. pnpm's default isolated linker does not hoist — it uses symlinks instead. Without `shamefully-hoist=true` in the root `.npmrc` (or the equivalent `node-linker=hoisted` in `pnpm-workspace.yaml`), Metro cannot find `react` and `react-native` in the places it expects, producing:

```
Error: Cannot find module 'react' from 'node_modules/react-native-system-scanner/lib/index.js'
```

This is silent at install time — `pnpm install` succeeds. The failure only surfaces when Metro starts.

**Why it happens:**
Metro's module resolver walks `node_modules` using Node.js resolution, which assumes flat/hoisted layout. pnpm's symlink structure is not flat; Metro's `watchFolders` must be configured to include the workspace root's `node_modules`, and `nodeModulesPaths` must point to the hoisted location.

**How to avoid:**
Add to `.npmrc` at the workspace root:
```
shamefully-hoist=true
```

And configure `example/metro.config.js`:
```js
const { getDefaultConfig } = require('@react-native/metro-config');
const path = require('path');
const root = path.resolve(__dirname, '..');

const config = getDefaultConfig(__dirname);
config.watchFolders = [root];
config.resolver.nodeModulesPaths = [
  path.resolve(__dirname, 'node_modules'),
  path.resolve(root, 'node_modules'),
];
module.exports = config;
```

`create-react-native-library` scaffolds this correctly for pnpm workspaces as of v0.62.0 — but verify the generated config rather than assuming it's correct.

**Warning signs:**
- `pnpm install` succeeds but `npx react-native start` throws module-not-found for `react` or `react-native`
- Works with yarn/npm but fails with pnpm
- `node_modules` at root contains only symlinks, not actual packages

**Phase to address:** Chunk 1 (Scaffold). Verify Metro starts correctly in `example/` before writing any native code.

---

### Pitfall 12: PDFKit Memory Spike on 15+ Page Scans — Jetsam Kills the Process

**What goes wrong:**
`PDFDocument` accumulates all page data in memory until `write(to:)` is called. VisionKit returns full-resolution `UIImage` objects per page (typical iPhone 14 Pro camera: 4032×3024 RGBA = ~47 MB uncompressed per image). A 15-page scan before JPEG pre-compression holds ~700 MB in RAM — well above the jetsam limit on older devices (iPhone X class: ~600 MB process limit). The OS kills the app silently with SIGKILL; no crash log appears in Xcode; the user sees the app restart.

**Why it happens:**
The `autoreleasepool { }` pattern in the PDF composition loop releases intermediate `UIImage` objects, but `PDFPage(image:)` internally retains the image data for the `PDFDocument` until `write(to:)` completes. Pre-compressing to JPEG before `PDFPage` creation is the critical step — it reduces per-page footprint from ~47 MB (RGBA) to ~1–3 MB (JPEG at 0.85).

**How to avoid:**
The v0.1 strategy (pre-compress before `PDFPage`) is correct but must be verified on device:

```swift
// PDFComposer.swift — correct pattern
func compose(images: [UIImage], to url: URL) throws {
    let pdfDoc = PDFDocument()
    for (index, image) in images.enumerated() {
        autoreleasepool {
            guard let jpegData = image.jpegData(compressionQuality: 0.85),
                  let compressedImage = UIImage(data: jpegData),
                  let page = PDFPage(image: compressedImage) else { return }
            pdfDoc.insert(page, at: index)
            // UIImage and jpegData released at end of autoreleasepool
        }
    }
    pdfDoc.write(to: url)
}
```

With pre-compression, each page holds ~1–3 MB in the `PDFDocument` → 15 pages ≈ 15–45 MB total — well within limits.

Test on real device with 15-page scan; profile in Instruments → Allocations to confirm peak memory stays below 150 MB for the composition step.

If OOM is still observed in v0.2, migrate to `CGPDFContextCreateWithURL` (writes page-by-page to disk with `CGPDFContextBeginPage`/`EndPage`) — documented as the v0.2 escape hatch in ARCHITECTURE.md.

**Warning signs:**
- App disappears without a crash report in Xcode Organiser (jetsam kill, not a crash)
- Instruments shows memory climbing steeply during composition and never releasing
- Consistent failure on 15+ page scans but not on 5-page scans

**Phase to address:** Chunk 2 (iOS native). Profile on device with 15-page scan before marking the phase done; document the soft limit (15 pages tested OK) in README.

---

### Pitfall 13: Android Edge-to-Edge (API 35+) and Predictive Back Break ML Kit Scanner UI

**What goes wrong:**
Android 15 (API 35) mandates edge-to-edge display for all apps targeting API 35+. The ML Kit scanner launches its own Activity (inside Play Services) — this Activity was built by Google and should handle edge-to-edge internally. However, if the host app sets `enableEdgeToEdge()` in `MainActivity` but does NOT propagate window inset configuration correctly, or if it sets `android:enableOnBackInvokedCallback="true"` in the manifest, the back gesture during scanning can misfire: the back gesture triggers `onBackPressed` in the host app's activity stack rather than in the ML Kit scanner Activity, cancelling the scan prematurely.

**Why it happens:**
`android:enableOnBackInvokedCallback="true"` at the application level opts ALL activities (including third-party ones launched via intent) into the new predictive back callback system. The ML Kit scanner Activity, being inside Play Services, may not handle this correctly for all Play Services versions.

**How to avoid:**
- Set `android:enableOnBackInvokedCallback="true"` at the **activity** level in the example app's `AndroidManifest.xml`, not at the `<application>` level, to avoid affecting the scanner Activity.
- Test on Android 15 emulator with predictive back enabled (it is mandatory in API 35+)
- In the module's README, note that consumers targeting API 35+ should scope `enableOnBackInvokedCallback` to their own activities

This is an **open question** from ARCHITECTURE.md (`startIntentSenderForResult` on API 35+). The `startIntentSenderForResult` method is still functional on API 35 — it was `startActivityForResult` that was deprecated in `ComponentActivity`. `startIntentSenderForResult` on `Activity` itself is not deprecated; it is the `ComponentActivity.startActivityForResult` wrapper that is. Verify by running on Android 15 emulator.

**Warning signs:**
- Back gesture during scan dismisses the scanner immediately without confirming (no cancel result code from ML Kit)
- `RESULT_CANCELED` received instead of expected completion after a swiped-back gesture mid-scan
- Only reproducible on API 35+ devices with predictive back enabled

**Phase to address:** Chunk 3 (Android native). Test on Android 15 API image before marking Android native work done.

---

### Pitfall 14: Codegen Spec Type Drift Between JS Types and Native Signatures

**What goes wrong:**
The codegen Spec in `NativeSystemScanner.ts` uses `string[]` for `outputFormats`. The native Android receives this as `ReadableArray` (not `List<String>`). If the Kotlin implementation casts it as `options.getString("outputFormats")` instead of `options.getArray("outputFormats")`, the cast fails at runtime with a null or ClassCastException that is not propagated cleanly — the JS promise never resolves.

Similarly on iOS: if the codegen generates `NSArray<NSString *>` but the Swift side reads it as a plain `NSArray`, the compiler accepts it but runtime type checks fail.

**Why it happens:**
Codegen translates TS types to native types, but the mapping is not always intuitive:
- `string[]` → `ReadableArray` on Android (not `List<String>`)
- `boolean` → `Bool` on iOS (but codegen may pass it as `NSNumber`)
- Optional fields (`maxPages?: number`) → `@Nullable Double` on Kotlin (not `Int?`) — ML Kit expects `Int`

**How to avoid:**
For each option field, document the codegen-to-native type mapping explicitly in a comment in `SystemScannerModule.kt`:

```kotlin
// outputFormats: ReadableArray (from string[] in spec) — use getArray()
val outputFormats = options.getArray("outputFormats") 
    ?: ReadableNativeArray().apply { /* default */ }

// maxPages: Double? (from number? in spec) — must convert to Int for ML Kit
val maxPages: Int? = if (options.hasKey("maxPages") && !options.isNull("maxPages"))
    options.getDouble("maxPages").toInt()
else null
```

Add a test in the example app that calls `scan()` with every combination of option fields — including null/omitted ones — and verifies the scan launches (not just that it doesn't crash at the options-parsing stage).

**Warning signs:**
- `scan()` fails silently (promise never resolves) when `maxPages` is set
- ClassCastException in LogCat when `outputFormats` is passed as `['jpeg', 'pdf']`
- Works with default options but fails with explicit options

**Phase to address:** Chunk 1 (Types) and Chunk 3 (Android native). Lock the type mapping in the spec design; validate all options in native before the module is marked done.

---

### Pitfall 15: ProGuard / R8 — ML Kit Consumer Rules Already Bundled, But Custom Kotlin Classes May Need Keep Rules

**What goes wrong:**
`play-services-mlkit-document-scanner` ships its own consumer ProGuard rules (via `proguard.txt` embedded in the AAR), so the ML Kit API classes themselves are safe under R8. However, if `SystemScannerModule.kt` or `FileCopyHelper.kt` is accessed via reflection (e.g., from a ProGuard-obfuscated host app), or if the library itself adds classes that are referenced only from `AndroidManifest.xml` or from codegen-generated stubs, R8 may strip them.

The specific risk: codegen generates a `SystemScannerPackage` class that is registered in `MainApplication.kt` by the consumer. If the consumer's R8 config doesn't keep `MainApplication` and its `getPackages()` return type, the package class may be stripped.

**Why it happens:**
RN library authors rarely test with `minifyEnabled true` + `shrinkResources true` in the consuming app. CI typically runs debug builds (`assembleDebug`). The issue only surfaces when a consumer does a release build with full R8.

**How to avoid:**
Add `consumer-rules.pro` to the library's `android/` directory:

```proguard
# Keep the TurboModule spec class (accessed by codegen-generated code)
-keep class com.systemscanner.SystemScannerModule { *; }
-keep class com.systemscanner.SystemScannerPackage { *; }
-keep class com.systemscanner.FileCopyHelper { *; }

# ML Kit consumer rules are bundled with play-services-mlkit-document-scanner;
# no additional ML Kit keep rules are needed.
```

Reference this file in `android/build.gradle`:
```groovy
android {
    defaultConfig {
        consumerProguardFiles 'consumer-rules.pro'
    }
}
```

**Warning signs:**
- Release APK crashes with `ClassNotFoundException` for `SystemScannerModule`
- Works in debug build, fails in release (classic R8 stripping symptom)
- Error appears in the consumer app's logcat, not in the library's CI (which only runs debug)

**Phase to address:** Chunk 3 (Android native). Add `consumer-rules.pro` in the same commit as `SystemScannerPackage.kt`.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Blocking `runBlocking` in `onActivityResult` for content:// copy | Simple, correct — no async lifecycle complexity | Hangs main thread for large scans (>10 pages, slow devices) | v0.1: always. Migrate to lifecycle-scoped coroutine in v0.2 if >500ms is measured |
| Hard-coded `JPEG` quality at 0.85 | No option surface | Consumer complaints if they need archival quality | v0.1 always; add `quality` option in v0.2 only if filed |
| `PDFDocument` (not CoreGraphics) for iOS PDF | High-level API; fewer lines of code | OOM on >15-page scans on older devices | v0.1 with documented page limit; migrate to CGPDFContext in v0.2 if OOM measured |
| Single `REQUEST_CODE` constant for ML Kit intent | Simple matching in `onActivityResult` | Breaks if consumer calls `scan()` concurrently (two pending promises, same request code) | Acceptable since `scan()` is serialized by the single `pendingPromise` guard |
| `shamefully-hoist` in pnpm | Metro just works | Negates pnpm's isolation benefits in the example/ workspace | Never for production monorepos; acceptable for this single library workspace |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| VisionKit presentation (iOS) | Using deprecated `UIApplication.shared.keyWindow` to find the presenter | `UIApplication.shared.connectedScenes` → `UIWindowScene` → `windows.first(where: { $0.isKeyWindow })` → walk `presentedViewController` chain |
| ML Kit `getStartScanIntent` (Android) | Treating all failures as `'unsupported'` | Inspect `MlKitException.errorCode`: `UNAVAILABLE` → `'model_unavailable'`; version mismatch → `'unsupported'` |
| Codegen spec to Kotlin type mapping | `options.getString("outputFormats")` for an array field | `options.getArray("outputFormats")` for `string[]`; `options.getDouble("maxPages").toInt()` for `number?` |
| PDFKit (iOS) | Creating `PDFPage(image: rawUIImage)` without JPEG compression | `image.jpegData(compressionQuality: 0.85)` → `UIImage(data:)` → `PDFPage(image:)` inside `autoreleasepool` |
| Expo config plugin | Importing from `@expo/config-plugins` directly | Import from `expo` — `const { withInfoPlist } = require('expo/config-plugins')` |
| pnpm workspace | Running `npm install` inside `example/` directly | Always `pnpm install` from workspace root; `example/` must not have its own lock file |
| iOS `PrivacyInfo.xcprivacy` | Omitting the `resource_bundles` podspec entry | Without `resource_bundles`, the plist is not included in the framework bundle and Apple's privacy scanner misses it |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| `PDFDocument` with uncompressed `UIImage` pages | Jetsam kill after scan; app restarts silently | Pre-compress every page to JPEG at 0.85 inside `autoreleasepool` before `PDFPage(image:)` | 8+ full-res pages on iPhone X; 15+ pages on iPhone 14 |
| `ContentResolver.openInputStream` on main thread without `Dispatchers.IO` | StrictMode DiskReadViolation in debug; ANR in release on slow devices | `withContext(Dispatchers.IO)` inside `runBlocking { }` in `onActivityResult` | Multi-page Android scans on slow eMMC storage (budget devices) |
| Duplicate module instances from `reactContext.addActivityEventListener` called multiple times | `onActivityResult` fires twice; promise resolved twice (RN throws on second resolve) | Register once in `init {}` block; check `pendingPromise == null` guard at top of `onActivityResult` | Whenever hot-reloading reconnects the RN bridge and re-initialises the module |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Writing scan files to `filesDir` (Android) or `Documents/` (iOS) instead of cache directories | Files persist indefinitely; user cannot clear them via OS storage settings; on Android `filesDir` is not automatically purged | Write to `cacheDir/scans/<ulid>/` (Android) and `NSCachesDirectory/scans/<ulid>/` (iOS) — OS-managed, app-private, purged under storage pressure |
| Returning `content://` URIs to JS (Android) instead of copying to `file://` | URIs expire with scanner Activity; worse, they grant temporary read permission that may be abused if the URI leaks | Always copy to `file://` in the library; never return `content://` to JS |
| Source maps published to npm leaking local file paths | Developer's home directory path (e.g., `/Users/joao/...`) visible in source maps on npm | Set `"sourceMaps": false` in `react-native-builder-bob` config, or add `lib/**/*.map` to `.npmignore`. Default in bob 0.41 does publish maps — verify with `npm pack --dry-run` |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Calling `scan()` without checking `isSupported()` on Huawei/no-GMS device | App appears to hang (promise stuck) or crashes | Show a disabled "Scan Document" button with tooltip when `isSupported()` returns false |
| Mapping ML Kit `UNAVAILABLE` as permanent `'unsupported'` error | User sees "device not supported" on a supported device that's just offline | Map `UNAVAILABLE` to `'model_unavailable'` with a "check internet and retry" message |
| Not calling `cleanup(scanId)` after upload | Storage accumulates — users report app using hundreds of MB | Document cleanup responsibility prominently; consider logging a warning to the console if >5 scan directories are present |
| iOS `scan()` resolving before the modal is fully dismissed | Consumer's UI mounts before the modal is gone → jarring layering | Resolve the promise inside the delegate callback (Apple calls this after the modal finishes dismissing), not after calling `dismiss()` |

---

## "Looks Done But Isn't" Checklist

- [ ] **iOS Swift bridge:** `react_native_system_scanner-Swift.h` import confirmed correct by running `pod install` and checking `DerivedData` — not just assumed from the podspec name
- [ ] **iOS Privacy manifest:** `PrivacyInfo.xcprivacy` included in `resource_bundles` in podspec and verified via Xcode's Product → Archive → Privacy Report
- [ ] **Android consumer rules:** `consumer-rules.pro` keeping `SystemScannerModule`, `SystemScannerPackage`, and `FileCopyHelper` is present and referenced in `build.gradle`
- [ ] **Android `pendingPromise` cleared on all paths:** Confirmed via code review that `pendingPromise = null` appears at the top of `onActivityResult` AND in the `addOnFailureListener` of `getStartScanIntent`
- [ ] **content:// copy timing:** Confirmed that `FileCopyHelper.copyResult` is invoked before `onActivityResult` returns (not in a detached coroutine launched after)
- [ ] **Source maps excluded from npm publish:** `npm pack --dry-run` shows no `*.map` files in the output
- [ ] **iOS multi-page memory:** Instruments profile on real device with 15-page scan shows peak <150 MB for PDF composition
- [ ] **Xcode version pinned in CI:** `.github/workflows/ci.yml` uses `uses: maxim-lobanov/setup-xcode@v1` with an explicit version pin, not the default
- [ ] **pnpm workspace resolution:** `example/` Metro starts cleanly (`react` and `react-native` resolve without warning) after a clean `pnpm install`
- [ ] **`isSupported()` tested on real Android without GMS:** Emulator with `default` or `aosp_atd` system image (not `google_apis`) returns `false` from `isSupported()` and `{ kind: 'error', code: 'unsupported' }` from `scan()`

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Wrong CRNL template chosen | HIGH (re-scaffold) | `rm -rf` the scaffolded repo, re-run `npx create-react-native-library@latest` with `turbo-module` template |
| Swift header name wrong | LOW | Correct the import in `.mm`, run `pod install` again, clean DerivedData |
| `pendingPromise` leaked (user reports hang) | MEDIUM | Add `LifecycleEventListener.onHostResume` timeout-reject; ship patch |
| `PrivacyInfo.xcprivacy` missing after App Store rejection | MEDIUM | Add the manifest, update podspec, release patch version, ask consumer to update |
| Source maps in published npm package | LOW | Add `*.map` to `.npmignore`, publish corrected patch |
| ProGuard stripping in consumer release build | HIGH (consumer discovery) | Add `consumer-rules.pro`, release patch; consumer must rebuild release |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Wrong CRNL template → old-arch shim | Chunk 1 (Scaffold) | Check `package.json` `codegenConfig.ios.type === "turbo"` immediately post-scaffold |
| Swift header name mismatch | Chunk 2 (iOS native) | First `xcodebuild` after pod install; inspect DerivedData for generated header |
| VisionKit delegate retain cycle | Chunk 2 (iOS native) | Instruments → Allocations: scan 3 times; verify `RNSystemScannerImpl` count stays at 1 |
| No presenter VC on cold launch | Chunk 2 (iOS native) | Test scenario: launch from notification with app in background state |
| `pendingPromise` leaked on Activity death | Chunk 3 (Android native) | Manual test: start scan, kill host app via ADB; verify next `scan()` rejects cleanly |
| `content://` copy deferred past Activity teardown | Chunk 3 (Android native) | 10-page scan on slow emulator; verify no `FileNotFoundException` in LogCat |
| `PrivacyInfo.xcprivacy` missing | Chunk 2 (iOS native) | Xcode Product → Archive → Privacy Report; `pod spec lint` should warn if absent |
| `model_unavailable` mapped as `unsupported` | Chunk 3 (Android native) | Test on real device with airplane mode + fresh GMS (no cached model) |
| `getEnforcing` module name mismatch | Chunk 1 (Scaffold) | App launch on iOS and Android without crash |
| ESM breaks consumer Jest | Chunk 6 (CI + Docs) | Document `transformIgnorePatterns` in README; verify with example app's Jest suite |
| pnpm Metro resolution failure | Chunk 1 (Scaffold) | `cd example && npx react-native start` starts without module-not-found errors |
| PDFKit memory spike >15 pages | Chunk 2 (iOS native) | Instruments profile on real device; 15-page scan must complete without jetsam |
| Edge-to-edge / predictive back on API 35+ | Chunk 3 (Android native) | Android 15 emulator test with back gesture during active scan |
| Codegen spec type drift | Chunk 1 (Types) + Chunk 3 (Android) | Example app: call `scan()` with all option combinations including edge cases |
| ProGuard stripping in release | Chunk 3 (Android native) | `consumer-rules.pro` present; run `assembleRelease` on example app and verify no `ClassNotFoundException` |
| Source maps leaking local paths | Chunk 6 (CI + Docs) | `npm pack --dry-run` shows no `.map` files in file list |
| `PrivacyInfo.xcprivacy` entries incomplete | Chunk 2 (iOS native) | Run `xcodebuild` archive and review Privacy Report; verify all required reason APIs declared |

---

## Open Questions from ARCHITECTURE.md — Expanded

### `model_unavailable` Detection on Android (ARCHITECTURE.md Risk, now addressed)

The ARCHITECTURE.md flagged detecting model unavailability as distinct from device unsupported. Verified:
- `MlKitException.UNAVAILABLE` (errorCode 14) is thrown by `getStartScanIntent().addOnFailureListener` when the model hasn't been downloaded
- `ModuleInstall.getClient(context).areModulesAvailable(scanner)` can be used as a preflight check inside `isSupported()`
- The `model_unavailable` error code in the `ScanErrorCode` union is confirmed correct and necessary

### PDFKit iOS 16+ `saveAllImagesAsJPEGKey` Fast Path (ARCHITECTURE.md Open Question)

iOS 16 added `PDFDocumentWriteOption.saveAllImagesAsJPEGKey` to force JPEG encoding during `write(to:)`. Since the deployment target is iOS 13+, this cannot be used unconditionally. Strategy:

```swift
func writeDocument(_ doc: PDFDocument, to url: URL) {
    if #available(iOS 16.0, *) {
        doc.write(to: url, withOptions: [.saveAllImagesAsJPEGKey: true])
    } else {
        // Pre-compressed pages already (JPEG data before PDFPage creation)
        doc.write(to: url)
    }
}
```

On iOS 16+, this option makes `write()` re-encode images as JPEG even if they were PNG. Combined with pre-compression (which we do), the write step is doubly safe. On iOS 13–15, pre-compression at `PDFPage` creation time is the only mitigation. Both paths are handled in v0.1.

### `startIntentSenderForResult` on API 35+ (ARCHITECTURE.md Open Question)

`Activity.startIntentSenderForResult()` is NOT deprecated in the Android SDK as of API 35. The deprecation affects `ComponentActivity.startActivityForResult()` (from `androidx.activity`). `startIntentSenderForResult` on `android.app.Activity` remains the correct API for launching `IntentSender`-based flows (like ML Kit). The `ActivityResultContracts.StartIntentSenderForResult` contract (the modern alternative) requires registering before the Activity starts — not possible in a plain TurboModule. The `ActivityEventListener` + `startIntentSenderForResult` pattern remains correct and non-deprecated for this use case.

---

## Sources

- `MlKitException` error codes — https://developers.google.com/android/reference/com/google/mlkit/common/MlKitException (verified May 2026)
- ML Kit known issues (CAPTURE_MODE_MANUAL bug) — https://github.com/googlesamples/mlkit/issues/846
- ML Kit document scanner initialization error on specific devices — https://github.com/googlesamples/mlkit/issues/823
- react-native-builder-bob ESM support documentation — https://oss.callstack.com/react-native-builder-bob/esm (verified May 2026)
- react-native-builder-bob v0.40.0 changelog — https://github.com/callstack/react-native-builder-bob/releases/tag/react-native-builder-bob@0.40.0
- pnpm + React Native Metro resolution — https://lorefnon.me/2024/01/29/using-react-native-metro-in-pnpm-workspace
- Apple Privacy Manifest enforcement — https://bitrise.io/blog/post/enforcement-of-apple-privacy-manifest-starting-from-may-1-2024
- Apple privacy manifest official docs — https://developer.apple.com/documentation/bundleresources/privacy-manifest-files
- Expo privacy manifest guide — https://docs.expo.dev/guides/apple-privacy/
- GitHub Actions Xcode support policy change — https://github.com/actions/runner-images/issues/12541
- Android emulator runner (AOSP vs Google APIs images) — https://github.com/ReactiveCircus/android-emulator-runner
- Android predictive back gesture — https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture
- Android 15 edge-to-edge enforcement — https://developer.android.com/codelabs/edge-to-edge
- `ActivityResultContracts.StartIntentSenderForResult` — https://developer.android.com/reference/androidx/activity/result/contract/ActivityResultContracts.StartIntentSenderForResult
- RN TurboModuleRegistry.getEnforcing issues — https://github.com/facebook/react-native/issues/50694
- UIWindowScene root VC access (iOS 13+) — https://sarunw.com/posts/how-to-get-root-view-controller/
- PDFKit memory issues — https://developer.apple.com/forums/thread/97515
- iOS jetsam memory limits — https://developer.apple.com/documentation/xcode/identifying-high-memory-use-with-jetsam-event-reports
- Production ML Kit document scanner (content:// copy timing) — https://gaikwadchetan93.medium.com/building-a-production-ready-document-scanner-in-android-with-google-ml-kit-2e76a82b99c5
- react-native-directory newArchitecture field — https://github.com/react-native-community/directory (libraries.json schema)
- ProGuard/R8 in multi-module Android projects — https://drjansari.medium.com/mastering-proguard-in-android-multi-module-projects-agp-8-4-r8-and-consumable-rules-ae28074b6f1f

---

*Pitfalls research for: react-native-system-scanner (TurboModule wrapping VisionKit + ML Kit)*
*Researched: 2026-05-04*
