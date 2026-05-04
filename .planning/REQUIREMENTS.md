# Requirements: react-native-system-scanner

**Defined:** 2026-05-04
**Core Value:** A 5-line `scan()` call that opens the OS-native document scanner and returns a stable result (`success` with image and/or PDF URIs, `cancelled`, or `error`) — without the consumer reasoning about cache eviction, permissions, or platform divergence.

## v1 Requirements

Requirements for the v0.1.0 ship. Each maps to one phase in the roadmap.

### Public API (JS)

- [ ] **API-01**: User can call `scan(options?: ScanOptions): Promise<ScanResult>` from JavaScript and receive a discriminated-union result
- [ ] **API-02**: `ScanResult` is a discriminated union: `{ kind: 'success', images?, pdfUri?, pageCount }` | `{ kind: 'cancelled' }` | `{ kind: 'error', code, message }`
- [ ] **API-03**: User cancellation resolves `{ kind: 'cancelled' }` — never throws or rejects
- [ ] **API-04**: `ScanOptions.outputFormats: ('jpeg' | 'pdf')[]` selects which formats native generates (default `['jpeg']`); requesting `['jpeg', 'pdf']` returns both
- [ ] **API-05**: `ScanOptions.maxPages?: number` is passed through to ML Kit on Android (clamped to ML Kit's internal cap); documented as no-op on iOS
- [ ] **API-06**: `ScanOptions.scannerMode?: 'full' | 'base'` (Android only) maps to `SCANNER_MODE_FULL` / `SCANNER_MODE_BASE`; iOS ignores; default `'full'`
- [ ] **API-07**: `ScanOptions.galleryImportAllowed?: boolean` (Android only) maps to `setGalleryImportAllowed`; iOS ignores; default `true`
- [ ] **API-08**: User can call `isSupported(): Promise<boolean>` to check device capability — returns `false` for Mac Catalyst, simulators, no-GMS Android, GMS too old; documented as not equivalent to camera-permission status
- [ ] **API-09**: User can call `cleanup(scanId?: string): Promise<void>` — with no args clears the entire `<libCacheDir>/scans/` tree; with `scanId` clears only that scan's folder
- [ ] **API-10**: All public symbols are exported from `index.ts` with full TSDoc; literal unions are used (no string-literal enums)
- [ ] **API-11**: A ULID is generated in JavaScript (via `ulidx`) per `scan()` call and passed to native as `scanId`, so the URI path is deterministic before native runs
- [ ] **API-12**: The full error code taxonomy is `'unsupported' | 'permission_denied' | 'unavailable' | 'model_unavailable' | 'cancelled_by_system' | 'unknown'` and every native failure is mapped to exactly one code
- [ ] **API-13**: Returned `images` are `file://` URIs (not absolute paths, not base64); `pdfUri` is a `file://` URI

### iOS Native (VisionKit)

- [ ] **IOS-01**: TurboModule scaffold uses Swift implementation (`SystemScanner.swift` with `@objc public`) bridged via a thin Obj-C++ `.mm` file generated from codegen
- [ ] **IOS-02**: `VNDocumentCameraViewController` is presented modally from the current key window's top view controller; delegate fires success/cancel/error
- [ ] **IOS-03**: `isSupported()` checks `VNDocumentCameraViewController.isSupported`
- [ ] **IOS-04**: When the caller requests `'pdf'`, scanned `UIImage`s are composed into a multi-page PDF via PDFKit, JPEG-compressed at 0.85 inside `autoreleasepool` per page
- [ ] **IOS-05**: Native writes outputs (`page-N.jpg` images and/or `scan.pdf`) into `<NSCachesDirectory>/com.lexxen.system-scanner/scans/<scanId>/` and resolves with the `file://` URIs
- [ ] **IOS-06**: `ios/PrivacyInfo.xcprivacy` is shipped, declaring `NSPrivacyAccessedAPICategoryFileTimestamp` (C617.1) and `NSPrivacyAccessedAPICategoryDiskSpace` (85F4.1); referenced via `resource_bundles` in the podspec
- [ ] **IOS-07**: Camera permission denial is detected (presentation fails with permission error) and surfaced as `{ kind: 'error', code: 'permission_denied' }`
- [ ] **IOS-08**: Memory-safe multi-page handling — images are released after each PDF page is appended; verified by Instruments on a real device with a ≥15-page scan without OS jetsam termination
- [ ] **IOS-09**: Swift uses `async/await` where the iOS deployment target allows; no singletons unless Apple's API forces them

### Android Native (ML Kit)

- [ ] **AND-01**: TurboModule is implemented in Kotlin; coroutines for async work; no Java; no RxJava
- [ ] **AND-02**: Scanner is launched via `ActivityEventListener` + `startIntentSenderForResult` (NOT `ActivityResultLauncher`, which is unsafe in plain TurboModules under new-arch); `reactContext.addActivityEventListener(this)`
- [ ] **AND-03**: `isSupported()` uses `ModuleInstall.areModulesAvailable()` to confirm ML Kit Document Scanner is present and the GMS version is sufficient
- [ ] **AND-04**: `GmsDocumentScannerOptions` is built from `ScanOptions` (page limit, scanner mode, gallery import, output format includes PDF when requested)
- [ ] **AND-05**: `content://` URIs returned by the scanner are copied **synchronously inside `onActivityResult`** (`runBlocking { withContext(Dispatchers.IO) { ... } }`) to `<cacheDir>/scans/<scanId>/` before resolving the JS promise
- [ ] **AND-06**: Output files written to `<context.cacheDir>/scans/<scanId>/`: `page-N.jpg` for images and `scan.pdf` for PDF; resolved as `file://` URIs
- [ ] **AND-07**: `MlKitException.UNAVAILABLE` (errorCode 14) is mapped to `'model_unavailable'` (model not yet downloaded), distinct from `'unsupported'` (GMS missing/too old)
- [ ] **AND-08**: A `consumer-rules.pro` file ships with appropriate ProGuard/R8 keep rules so consuming apps don't need to add their own
- [ ] **AND-09**: Stored `pendingPromise` is rejected cleanly if the host Activity is destroyed before `onActivityResult` fires — no leaked promises
- [ ] **AND-10**: Verified to work correctly under Android 15+ predictive back gestures and edge-to-edge display

### Expo Config Plugin

- [ ] **PLUG-01**: A config plugin at `plugin/` (sub-path-exported as `react-native-system-scanner/plugin`) supports `cameraPermission?: string` and writes `NSCameraUsageDescription` to `Info.plist` via `withInfoPlist`
- [ ] **PLUG-02**: The plugin adds `implementation("com.google.android.gms:play-services-mlkit-document-scanner:16.0.0")` to the consumer app's `build.gradle` via `withAppBuildGradle`
- [ ] **PLUG-03**: The plugin supports an opt-in `mlKitModelInstallTime?: boolean` flag that, when true, adds the `com.google.mlkit.vision.DEPENDENCIES` `<meta-data>` to `AndroidManifest.xml` for install-time model bundling
- [ ] **PLUG-04**: The README documents manual setup steps for non-Expo consumers (Info.plist + Gradle dep)

### Quality Gate (cross-cutting)

- [ ] **QUAL-01**: TypeScript strict mode, zero `any` in shipped code; verified by `tsc --strict --noEmit`
- [ ] **QUAL-02**: Every exported symbol carries TSDoc with at least a one-line summary
- [ ] **QUAL-03**: No `console.log` or equivalent in shipped code (lint-enforced via Biome)
- [ ] **QUAL-04**: Error messages are actionable: include the version detected when relevant and a docs URL (e.g., `"Play Services 21+ required for ML Kit Document Scanner. Detected: 19.8.39. See: https://..."`)
- [ ] **QUAL-05**: No silent failures — every native error path resolves with a typed `'error'` result

### Example App

- [ ] **EX-01**: `example/` is a pnpm workspace member that consumes the local library
- [ ] **EX-02**: Demonstrates image-only output (`outputFormats: ['jpeg']`)
- [ ] **EX-03**: Demonstrates PDF-only output (`outputFormats: ['pdf']`)
- [ ] **EX-04**: Demonstrates mixed output (`outputFormats: ['jpeg', 'pdf']`)
- [ ] **EX-05**: Demonstrates cancellation handling — UI shows a non-error "Cancelled" state when the user backs out
- [ ] **EX-06**: Demonstrates graceful error handling on an unsupported device (no GMS Android emulator, simulator iOS) — surfaces the typed error code and message
- [ ] **EX-07**: Tested on real iOS hardware, real Android hardware, and an AOSP/no-GMS Android emulator
- [ ] **EX-08**: A `TESTING.md` documents the manual test matrix (3 output formats × {iOS, Android, Android-no-GMS} × {success, cancel, denied} = 27 cases) with checkboxes; ship requires all-pass

### CI

- [ ] **CI-01**: GitHub Actions workflow runs on PR and on push to main
- [ ] **CI-02**: TypeScript check (`tsc --noEmit`) is green
- [ ] **CI-03**: Biome lint + format check is green
- [ ] **CI-04**: JS unit tests run and pass
- [ ] **CI-05**: iOS pod install + `xcodebuild` builds the example app on a pinned macOS runner / Xcode version
- [ ] **CI-06**: Android `./gradlew assembleDebug` builds the example app on Linux runner

### Release & Distribution

- [ ] **REL-01**: Package is named `react-native-system-scanner` (does not collide with `react-native-document-scanner-plugin`); name signals "system UI"
- [ ] **REL-02**: `package.json` peer dependencies pin `react-native >=0.82.0` (new-arch mandatory; documented in README)
- [ ] **REL-03**: `package.json` `files` field is set so `npm pack --dry-run` includes only `lib/`, `ios/`, `android/`, `plugin/`, `src/`, `*.podspec`, `README.md`, `CHANGELOG.md`, `LICENSE`, and the privacy manifest — and excludes source maps
- [ ] **REL-04**: README's first screen contains: install command, 5-line minimal example, supported platforms/versions, and a demo gif — under 60 seconds to comprehend
- [ ] **REL-05**: `CHANGELOG.md`, `LICENSE` (MIT), `CONTRIBUTING.md` are present
- [ ] **REL-06**: `v0.1.0` is published to npm
- [ ] **REL-07**: GitHub release `v0.1.0` is tagged with the demo gif attached
- [ ] **REL-08**: Library is submitted to react-native-directory and Expo's third-party module list

## v2 Requirements

Deferred to a future release. Tracked but not in the v0.1.0 roadmap.

### API expansion

- **V2-01**: `jpegQuality?: number` option on `scan()` (hard-coded 0.85 in v0.1)
- **V2-02**: Android `captureMode: 'auto' | 'manual'` (currently implied by `scannerMode`)
- **V2-03**: Per-page metadata in the success result (page index, original orientation)
- **V2-04**: PDF metadata options (title, author)

### Performance

- **V2-05**: PDFDocumentWriteOption.saveAllImagesAsJPEGKey fast path on iOS 16+
- **V2-06**: Streaming/incremental PDF write for very large scans (>30 pages)

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Custom scanner UI | Contradicts the library's premise (system UI fidelity); would balloon binary size by ~30 MB |
| OpenCV / Vision Camera fallback for unsupported devices | Returning `{ kind: 'error', code: 'unsupported' }` is the v0.1 answer; OpenCV inflates scope and binary size |
| Old-architecture (bridge) support | Doubles surface area; contradicts the new-arch learning goal; RN ≥0.82 makes new-arch mandatory anyway |
| Post-scan editing (crop, rotate, filter) | Both system UIs include their own editing UI; duplicating is worse UX |
| OCR / text extraction | Different API surface, separate library scope |
| Barcode / QR scanning | Owned by `react-native-vision-camera` and `expo-barcode-scanner` |
| Continuous / batch scanning beyond system UI native support | System constraints are the constraints |
| View component / embedded camera surface | Both platform APIs are exclusively modal — nothing to embed |
| iPad-specific presentation tweaks beyond OS defaults | OS defaults are acceptable; verify split-view works and ship |
| Base64 image output | Causes OOM for multi-page; `file://` URIs are correct for large binaries |
| `isSupported()` returning camera-permission status | Mixing capability and permission creates confusing semantics; permission is a runtime check exposed as the `'permission_denied'` error code |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| API-01 through API-13 | TBD | Pending |
| IOS-01 through IOS-09 | TBD | Pending |
| AND-01 through AND-10 | TBD | Pending |
| PLUG-01 through PLUG-04 | TBD | Pending |
| QUAL-01 through QUAL-05 | TBD | Pending |
| EX-01 through EX-08 | TBD | Pending |
| CI-01 through CI-06 | TBD | Pending |
| REL-01 through REL-08 | TBD | Pending |

**Coverage:**
- v1 requirements: 58 total
- Mapped to phases: 0 (filled by roadmapper)
- Unmapped: 58 ⚠️ — populated during roadmap creation

---
*Requirements defined: 2026-05-04*
*Last updated: 2026-05-04 after initialization*
