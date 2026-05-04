# Project Research Summary

**Project:** react-native-system-scanner
**Domain:** React Native TurboModule library wrapping platform-native modal document scanner UIs
**Researched:** 2026-05-04
**Confidence:** HIGH

## Executive Summary

`react-native-system-scanner` wraps Apple's `VNDocumentCameraViewController` (VisionKit, iOS 13+) and Google's `GmsDocumentScanning` (ML Kit, Play Services) — the OS-shipped document scanner UIs that existing RN libraries (OpenCV-based, old-arch-only, unmaintained) do not yet wrap. The pattern is a single TurboModule with a three-function public API (`scan`, `isSupported`, `cleanup`), new-architecture only, Swift + Kotlin native, with a JS wrapper layer that generates a ULID per call, passes it to native for stable file path generation, and promotes the flat native result to a typed discriminated union. This is a well-understood pattern modelled on `expo-image-picker` for Android wiring and the Expo team's Swift TurboModule bridging idiom for iOS.

The recommended build order is six sequential chunks — scaffold and lock types first (all native work depends on these), iOS and Android native in parallel after types are locked, Expo config plugin in parallel with native, example app once both platforms work, then CI/docs/ship. The most dangerous phases are Android native (two critical-severity pitfalls: `content://` URI expiry and `pendingPromise` lifecycle) and iOS native (delegate retain cycle, VisionKit presenter safety, and the mandatory `PrivacyInfo.xcprivacy` that blocks App Store submission). All pitfalls have established mitigations from sources verified against official docs.

The primary risk is time: this is a 2-3 week scope with non-trivial native wiring on both platforms. The mitigation is strict scope discipline — the anti-feature list in FEATURES.md is as load-bearing as the feature list. The secondary risk is platform-specific correctness on Android (ML Kit `UNAVAILABLE` vs `unsupported`, predictive back on API 35+, `content://` copy timing) where subtle distinctions produce wrong consumer-visible behaviour without any build-time error.

## Key Findings

### Recommended Stack

The library is scaffolded with `create-react-native-library@0.62.0` using the `turbo-module` (not `turbo-module-mixed`) template, Kotlin + Obj-C language selection, pnpm workspace manager. Post-scaffold, Swift is wired manually via the established Expo-team pattern: thin `.mm` bridge delegates every method to a `@objc public class` in `.swift`. All iOS logic lives in Swift; the `.mm` file is ~20 lines of glue. Biome 2.4.x replaces ESLint + Prettier. `react-native-builder-bob@0.41.0` produces ESM-only output. `ulidx@2.4.1` generates scan IDs in the JS layer.

**Core technologies:**
- React Native `>=0.82` (peer floor) — 0.82 is when new-arch became mandatory; `>=0.74` (PROJECT.md) is too low; earlier versions require opt-in opt-in, making the peer signal ambiguous.
- TypeScript `~5.8` (devDep) — TS 6.0 (Mar 2026) has breaking module/strict changes not yet confirmed compatible with RN codegen; hold at 5.8.
- VisionKit `VNDocumentCameraViewController` + PDFKit — iOS system frameworks, zero binary impact; PDFKit composes `[UIImage]` to PDF since VisionKit returns images only.
- `com.google.android.gms:play-services-mlkit-document-scanner:16.0.0` — ML Kit in Play Services; zero APK impact (models live in Play Services, on-demand by default).
- `ulidx@2.4.1` — JS-side ULID generation passed to native as `scanId` string; native writes to `<cacheDir>/scans/<scanId>/`.
- Biome `2.4.14`, pnpm `>=9`, `react-native-builder-bob@0.41.0` — tooling matching MMKV/Reanimated house style.

### Expected Features

**Must have (table stakes):**
- `scan(options)` returning discriminated-union `{ kind: 'success' | 'cancelled' | 'error' }` — cancellation is not a thrown exception.
- `isSupported()` — capability check (not permission check); gate for disabled UI on Huawei/simulator.
- `cleanup(scanId?)` — explicit storage management; library-owned cache means library owns cleanup.
- Stable `file://` URIs — native copies bytes to `<libCacheDir>/scans/<scanId>/` before resolving; Android `content://` URIs must never reach JS.
- Output format choice: `'jpeg'`, `'pdf'`, or both — iOS PDFKit composition; Android native PDF via `GmsDocumentScanningResult.pdf`.
- Typed error codes with actionable messages — `'unsupported'`, `'permission_denied'`, `'unavailable'`, `'model_unavailable'`, `'unknown'`.
- Expo config plugin — `NSCameraUsageDescription`, Gradle dependency injection, `mlKitModelInstallTime` flag.
- TypeScript strict, zero `any`, TSDoc on all exported symbols.

**Should have (differentiators vs OpenCV alternatives):**
- Android `scannerMode: 'full' | 'base'` — exposes `SCANNER_MODE_FULL` (ML image cleaning) which OpenCV libs cannot offer.
- Android `galleryImportAllowed: boolean` — exposes `setGalleryImportAllowed`; iOS ignores (no API).
- `maxPages` — Android passes to `setPageLimit()` (~25 internal cap); iOS ignores (document as no-op).
- `model_unavailable` as a distinct error code from `'unsupported'` — device has GMS but model not yet downloaded; consumer shows "retry" not "unsupported device".

**Defer (v2+):**
- JPEG quality option — 0.85 hardcoded; add only if filed as issue.
- Android `captureMode` (auto vs manual) — `SCANNER_MODE_FULL` already implies auto-capture.
- CoreGraphics PDF composition — PDFKit sufficient for v0.1 with pre-compression; migrate only if OOM is measured on device.

**The `scan()` API surface (complete, for requirements derivation):**

```typescript
interface ScanOptions {
  outputFormats?: ('jpeg' | 'pdf')[];   // default: ['jpeg']
  maxPages?: number;                     // Android: setPageLimit (~25 cap); iOS: no-op
  scannerMode?: 'full' | 'base';        // Android only; default: 'full'
  galleryImportAllowed?: boolean;        // Android only; default: true
}

type ScanResult =
  | { kind: 'success'; images?: string[]; pdfUri?: string; pageCount: number }
  | { kind: 'cancelled' }
  | { kind: 'error'; code: ScanErrorCode; message: string };

type ScanErrorCode =
  | 'unsupported'        // VisionKit isSupported=false, or GMS absent/too old
  | 'permission_denied'  // Camera denied at scan() time
  | 'unavailable'        // Scanner failed to launch
  | 'model_unavailable'  // ML Kit model not downloaded (Android, offline)
  | 'unknown';
```

`images` and `pdfUri` are optional — present only when the corresponding format was requested. `pageCount` is always present on success (useful when only `pdfUri` was requested).

### Architecture Approach

Six files form the JS layer (`index.ts`, `scan.ts`, `NativeSystemScanner.ts`, `types.ts`), with iOS native split into a thin `.mm` bridge + `RNSystemScannerImpl.swift` + `PDFComposer.swift` (isolated for the CoreGraphics v0.2 swap), and Android native as `SystemScannerModule.kt` (TurboModule + `ActivityEventListener`) + `ScannerLauncher.kt` + `FileCopyHelper.kt` + `SystemScannerPackage.kt`. An Expo config plugin lives in `plugin/src/index.ts` as a sub-path export. The codegen spec (`NativeSystemScanner.ts`) is the single source of truth between JS types and native signatures.

**Major components:**
1. `src/scan.ts` — Generates ULID, normalises options with defaults, calls TurboModule, promotes flat native result to typed discriminated union.
2. `ios/RNSystemScannerImpl.swift` — All VisionKit + PDFKit logic; `@objc public class` delegated to from thin `.mm` bridge.
3. `ios/PDFComposer.swift` — Isolated `[UIImage]` to `PDFDocument` composition with per-page `autoreleasepool`; designed as a single-file swap for CoreGraphics in v0.2.
4. `android/SystemScannerModule.kt` — TurboModule implementing `ActivityEventListener`; stores `pendingPromise`; dispatches to ML Kit and handles `onActivityResult`.
5. `android/FileCopyHelper.kt` — Copies `content://` URIs to `cacheDir/scans/<scanId>/` synchronously inside `onActivityResult`; also runs GMS version check.
6. `plugin/src/index.ts` — Config plugin: `withInfoPlist` (camera permission), `withAppBuildGradle` (Gradle dep), `withAndroidManifest` (model timing flag).

### Critical Pitfalls

1. **Wrong CRNL template (`turbo-module-mixed` instead of `turbo-module`)** — Silently produces old-arch shim; verify `codegenConfig.ios.type === "turbo"` immediately post-scaffold. Re-scaffold is the only recovery.

2. **`content://` URI copy deferred to a detached coroutine (Android)** — ML Kit's `content://` URIs expire when the scanner Activity finishes. Copy must happen inside `onActivityResult` using `runBlocking { withContext(Dispatchers.IO) { ... } }`. Any `launch { }` that suspends past scanner process teardown causes `FileNotFoundException`.

3. **`pendingPromise` leaked on Activity destruction mid-scan (Android)** — Phone call, system kill, or RAM pressure destroys the host Activity without firing `onActivityResult`. Implement `LifecycleEventListener.onHostResume` with a timeout-reject. Always set `pendingPromise = null` at the top of `onActivityResult` before branching.

4. **`PrivacyInfo.xcprivacy` missing (iOS)** — Required since May 2024 for any SDK using file timestamp or disk space APIs (`NSCachesDirectory` triggers both). App Store submission is rejected with `ITMS-91053`. Must be added in `ios/` with a `resource_bundles` podspec entry. `create-react-native-library` does not scaffold this.

5. **`ML Kit UNAVAILABLE` mapped as `'unsupported'`** — `MlKitException.UNAVAILABLE` (errorCode 14) fires when the model has not downloaded (device has GMS but is offline). Mapping it as `'unsupported'` shows "device not supported" to a user who just needs an internet connection. Inspect `errorCode` in `addOnFailureListener`; map to `'model_unavailable'` with a retry message.

6. **VisionKit delegate retain cycle** — `RNSystemScannerImpl` as VisionKit delegate must declare `weak var documentCamera`; nil out `pendingResolve`/`pendingReject` in all three delegate callback paths. Verify with Instruments Allocations across 3 successive scans.

## Implications for Roadmap

The build chunks from ARCHITECTURE.md are the phase template. The dependency graph is hard: types must be locked before native work; iOS and Android are independent after that; example app requires both platforms; CI/ship requires everything.

### Phase 1: Scaffold + Types + JS Skeleton (Chunk 1)

**Rationale:** All other phases depend on the TypeScript types and codegen spec being stable. Native implementations are generated from the spec; changing the spec after Chunk 2 or 3 starts requires regenerating stubs on both platforms. Lock `ScanOptions`, `ScanResult`, and `ScanErrorCode` before any native work begins.

**Delivers:** Working npm package shape with correct public API; `scan()` returns `{ kind: 'error', code: 'unknown' }` stub; Biome + TSC passing; Jest tests for pure-TS logic; pnpm workspace with Metro starting cleanly.

**Addresses:** `scan()` API surface, result types, error codes, TypeScript strict/zero-`any` requirement, ULID dependency.

**Avoids:** Wrong CRNL template (verify immediately), module name mismatch across the three registration points, pnpm Metro resolution failure, codegen spec type drift (lock mapping comments now).

**Research flag:** Standard patterns — no additional research needed.

### Phase 2: iOS Native — VisionKit + PDFKit (Chunk 2)

**Rationale:** iOS is lower risk than Android for Activity lifecycle complexity; starting here builds confidence before the harder Android wiring. PDFKit memory behaviour must be profiled on real hardware before marking done.

**Delivers:** `scan()` functional on real iOS device for all three output formats (`jpeg`, `pdf`, both); stable `file://` URIs; `isSupported()` returning false on simulator; `cleanup()` working.

**Implements:** `RNSystemScannerImpl.swift`, `PDFComposer.swift`, thin `.mm` bridge, `PrivacyInfo.xcprivacy`.

**Avoids:** Swift header name mismatch (check DerivedData after first `pod install`), VisionKit delegate retain cycle (Instruments profile), missing presenter VC on cold launch (`topMostViewController()` guard), PDFKit memory spike (pre-compress all pages, `autoreleasepool` per page, profile 15-page scan), missing `PrivacyInfo.xcprivacy` (add in same commit as Swift implementation).

**Research flag:** Standard patterns — VisionKit, PDFKit, and the Swift TurboModule bridging pattern are all well-documented. No research phase needed.

### Phase 3: Android Native — ML Kit + ActivityEventListener (Chunk 3)

**Rationale:** The two highest-severity runtime correctness pitfalls for the entire project live here. This phase requires the most careful implementation and the most thorough device testing.

**Delivers:** `scan()` functional on real Android device and GMS emulator; `isSupported()` returning false on no-GMS emulator; correct `'model_unavailable'` vs `'unsupported'` distinction; stable `file://` URIs; ProGuard consumer rules.

**Implements:** `SystemScannerModule.kt` + `ActivityEventListener`, `ScannerLauncher.kt`, `FileCopyHelper.kt`, `SystemScannerPackage.kt`, `consumer-rules.pro`.

**Avoids:** `content://` copy deferred past Activity teardown (`runBlocking` in `onActivityResult`), `pendingPromise` leak on Activity death (`LifecycleEventListener` + timeout-reject), `model_unavailable` mapped as `'unsupported'` (inspect `MlKitException.errorCode`), ProGuard stripping in release (add `consumer-rules.pro` now), predictive back on API 35+ (test on Android 15 emulator), codegen type drift (`ReadableArray`/`Double?` mappings documented in comments).

**Research flag:** `ActivityEventListener` vs `ActivityResultLauncher` decision is resolved. `startIntentSenderForResult` confirmed non-deprecated on API 35. No research phase needed, but Android 15 emulator test is mandatory before marking done.

### Phase 4: Expo Config Plugin (Chunk 4)

**Rationale:** Can begin as soon as the Gradle dependency version is known from Chunk 3. Does not affect runtime behaviour; low complexity, low risk.

**Delivers:** One-line Expo plugin setup for `NSCameraUsageDescription`, `play-services-mlkit-document-scanner` Gradle dep, and `mlKitModelInstallTime` flag.

**Implements:** `plugin/src/index.ts` with `withInfoPlist`, `withAppBuildGradle`, `withAndroidManifest`; sub-path export in `package.json`.

**Avoids:** Importing from `@expo/config-plugins` directly (import from `expo` instead); declaring `expo-modules-core` as a peer (use `expo: ">=56.0.0"` optional peer only).

**Research flag:** Standard patterns — exact `expo-image-picker` config plugin patterns apply. No research phase needed.

### Phase 5: Example App + Test Matrix (Chunk 5)

**Rationale:** Integration proof. Must demonstrate every documented behaviour including the no-GMS failure path, which requires an AOSP emulator image (not Google APIs).

**Delivers:** `App.tsx` demonstrating image-only, PDF-only, both formats, cancel, and unsupported-device error; `TESTING.md` with 3 output formats x {iOS, Android, Android-no-GMS} x {success, cancel, permission-denied} = 27 documented test cases.

**Avoids:** Testing only on GMS emulator (must also test AOSP image for no-GMS path); testing only happy path; skipping real-device PDF memory test.

**Research flag:** Standard patterns — no research phase needed.

### Phase 6: CI + Docs + Ship (Chunk 6)

**Rationale:** Final gate. Green CI on all four jobs before any publish step.

**Delivers:** GitHub Actions CI (TSC, Biome, Jest, `pod install + xcodebuild`, `assembleDebug`); `README.md` first-screen comprehension in <60 seconds; `CHANGELOG.md`, `LICENSE`, `CONTRIBUTING.md`, `TESTING.md`; `v0.1.0` on npm; GitHub release with demo gif; submitted to react-native-directory and Expo third-party list.

**Avoids:** Source maps in npm publish (`npm pack --dry-run` before publish); missing `transformIgnorePatterns` guidance for consumer Jest; CI using default Xcode version (pin with `maxim-lobanov/setup-xcode@v1`); Node < 20.19 in CI (`react-native-builder-bob@0.41` requires Node 20+).

**Research flag:** Standard patterns — no research phase needed.

### Phase Ordering Rationale

- **Types before native:** The codegen spec generates Kotlin and Obj-C++ stubs. Changing it mid-native-implementation forces regeneration on both platforms. Lock first.
- **iOS before Android:** iOS wiring is less lifecycle-complex; building iOS first validates the TurboModule plumbing before tackling the harder Android Activity lifecycle problems.
- **Config plugin in parallel with native:** Config plugin does not affect runtime; can be developed alongside Android native (Phase 3 and 4 overlap). Testing it requires the example app (Phase 5).
- **Example app gates ship:** The 27-case test matrix is the quality gate. No ship without documented test results.

### Cross-Cutting Risks

| Risk | Phases Affected | Mitigation |
|------|-----------------|------------|
| Type definitions locked in Phase 1 | All subsequent phases | Review full `ScanOptions`, `ScanResult`, `ScanErrorCode` shape before writing any native code |
| `PrivacyInfo.xcprivacy` iOS-only but blocks App Store for all consumers | Phase 2 (author), Phase 6 (distribution) | Create alongside Swift implementation; verify via Xcode Privacy Report before marking Phase 2 done |
| `content://` URI copy on Android is the most failure-prone runtime path | Phases 3 and 5 | `runBlocking` copy in `onActivityResult`; 10-page slow-emulator test in Phase 5 |
| RN peer floor `>=0.82` (not `>=0.74` as in PROJECT.md Constraints) | Phase 1 (`package.json`), Phase 6 (README) | Set correctly in `peerDependencies` from the start |
| `model_unavailable` is a distinct error code from `unsupported` | Phases 1 (types), 3 (impl), 5 (test), 6 (docs) | `ScanErrorCode` union must include `model_unavailable`; test with airplane mode + fresh GMS in Phase 5 |

### Research Flags

No phase needs a `/gsd:research-phase` call. All open questions from ARCHITECTURE.md were resolved during pitfalls research:
- `ActivityEventListener` vs `ActivityResultLauncher` — resolved: use `ActivityEventListener`.
- `startIntentSenderForResult` on API 35+ — resolved: not deprecated, correct API.
- `model_unavailable` detection — resolved: inspect `MlKitException.errorCode` in `addOnFailureListener`.
- PDFKit iOS 16+ `saveAllImagesAsJPEGKey` — resolved: use `#available(iOS 16.0, *)` conditional alongside pre-compression fallback.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions verified against official sources (Maven, npm, Apple docs, Google ML Kit docs) as of May 2026 |
| Features | HIGH | Platform APIs verified against official docs; competitor APIs directly inspected; `scan()` shape validated against both platform capabilities |
| Architecture | HIGH | File structure from CRNL template inspection; data flow from platform API docs; Android pattern from `expo-image-picker` source |
| Pitfalls | HIGH | Each pitfall grounded in official sources, issue trackers, or production community write-ups; no speculation |

**Overall confidence:** HIGH

### Gaps to Address

- **PDFKit memory ceiling:** The 15-page soft limit is theoretically derived. Must be validated empirically on a real iPhone during Phase 2. If jetsam kills occur below 15 pages, the CoreGraphics escape hatch must move to v0.1.
- **Android 15 predictive back:** The `startIntentSenderForResult` + `ActivityEventListener` path is theoretically correct on API 35 but must be verified on an Android 15 emulator in Phase 3.
- **ML Kit model download preflight:** `ModuleInstall.areModulesAvailable()` inside `isSupported()` is documented as the correct preflight; needs implementation and testing in Phase 3.

## Sources

### Primary (HIGH confidence)

- Apple VisionKit docs — `VNDocumentCameraViewController`, `VNDocumentCameraScan` (verified May 2026)
- Apple PDFKit docs — `PDFDocument`, `PDFPage`, iOS 16 `saveAllImagesAsJPEGKey` (verified May 2026)
- Google ML Kit Document Scanner — https://developers.google.com/ml-kit/vision/doc-scanner/android (updated Apr 22, 2026)
- `GmsDocumentScannerOptions` API reference (verified May 2026)
- `play-services-mlkit-document-scanner:16.0.0` Maven (verified May 2026)
- React Native 0.82 blog post — new-arch mandatory, bridge removed
- React Native 0.85 blog post — current stable (Apr 2026)
- `create-react-native-library@0.62.0` GitHub releases (Apr 2026)
- `react-native-builder-bob@0.41.0` changelog (Apr 2026)
- Biome 2.4.14 — https://biomejs.dev (May 2026)
- TypeScript 6.0 — breaking changes that rule it out for v0.1
- Expo config plugin peer dep guidance — https://docs.expo.dev/config-plugins/development-for-libraries/ (Apr 2, 2026)
- Apple Privacy Manifest enforcement — enforced May 1, 2024; ITMS-91053 rejection code confirmed
- `expo-image-picker` source — Android `ActivityEventListener` pattern (source inspection)

### Secondary (MEDIUM confidence)

- Swift TurboModule bridging pattern — community guide + Expo team usage confirmed
- PDFKit memory limits — Apple Developer Forums, iOS jetsam documentation
- `content://` URI copy timing — production ML Kit scanner community write-up

---
*Research completed: 2026-05-04*
*Ready for roadmap: yes*
