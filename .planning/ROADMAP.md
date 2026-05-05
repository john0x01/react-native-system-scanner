# Roadmap: react-native-system-scanner

## Overview

Six phases, derived directly from the architecture's build-chunk dependency graph. Phase 1 locks the TypeScript contract that every native implementation depends on. Phases 2 (iOS), 3 (Android), and 4 (Expo plugin) are independent after Phase 1 completes and can be developed in parallel by a single engineer working sequentially. Phase 5 integrates both platforms into the example app and gates ship. Phase 6 closes CI, docs, and distribution.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Scaffold + Types + JS Skeleton** - Working npm package shape with frozen public API, strict TypeScript, and stub native returning error
- [ ] **Phase 2: iOS Native (VisionKit + PDFKit)** - `scan()` functional on real iOS device; all three output formats return stable `file://` URIs
- [ ] **Phase 3: Android Native (ML Kit + ActivityEventListener)** - `scan()` functional on real Android device; correct error taxonomy; no leaked promises
- [ ] **Phase 4: Expo Config Plugin** - One-line Expo plugin setup for iOS permission, Android Gradle dep, and install-time model flag
- [ ] **Phase 5: Example App + Test Matrix** - Every documented behaviour demonstrated and manually verified across all 27 test cases
- [ ] **Phase 6: CI + Docs + Ship** - Green CI on all four jobs; README first-screen comprehension in under 60 seconds; v0.1.0 on npm

## Phase Details

### Phase 1: Scaffold + Types + JS Skeleton
**Goal**: The published package has the correct shape and frozen API surface; calling `scan()` returns a typed stub error; Biome and TSC are green; Metro starts in the example app
**Depends on**: Nothing (first phase)
**Requirements**: API-01, API-02, API-03, API-04, API-05, API-06, API-07, API-08, API-09, API-10, API-11, API-12, API-13, QUAL-01, QUAL-02, QUAL-03, QUAL-04, QUAL-05
**Success Criteria** (what must be TRUE):
  1. `import { scan, isSupported, cleanup } from 'react-native-system-scanner'` compiles under `tsc --strict --noEmit` with zero errors and zero `any`
  2. Calling `scan()` in the running example app returns `{ kind: 'error', code: 'unknown', message: 'Not implemented' }` — it does not throw and does not hang; `switch (result.kind)` narrows correctly in TypeScript
  3. `tsc --noEmit` and `biome check` both exit 0; no `console.log` survives the lint pass
  4. Every exported symbol (`scan`, `isSupported`, `cleanup`, `ScanOptions`, `ScanResult`, `ScanErrorCode`) has a TSDoc comment; literal unions are used, no string-literal enums
  5. Metro starts cleanly in `example/` after a fresh `pnpm install` with no module-not-found warnings for `react` or `react-native`
**Plans**: TBD

### Phase 2: iOS Native (VisionKit + PDFKit)
**Goal**: Users on a real iOS device can scan documents and receive stable `file://` URIs for images and/or a PDF; `isSupported()` returns false on simulator; `cleanup()` removes files; `PrivacyInfo.xcprivacy` ships
**Depends on**: Phase 1
**Requirements**: IOS-01, IOS-02, IOS-03, IOS-04, IOS-05, IOS-06, IOS-07, IOS-08, IOS-09
**Success Criteria** (what must be TRUE):
  1. On a real iPhone, `scan({ outputFormats: ['jpeg'] })` returns `{ kind: 'success', images: ['file:///...'], pageCount: N }` and each URI points to a readable JPEG file on disk after the scanner has been dismissed
  2. On a real iPhone, `scan({ outputFormats: ['pdf'] })` returns a `pdfUri` pointing to an openable multi-page PDF; `scan({ outputFormats: ['jpeg', 'pdf'] })` returns both fields
  3. A 15-page scan completes without OS process termination; Instruments Allocations profile confirms peak PDF composition memory under 150 MB
  4. `isSupported()` returns `false` on the iOS Simulator; backing out of the scanner modal returns `{ kind: 'cancelled' }`; denying camera permission returns `{ kind: 'error', code: 'permission_denied' }`
  5. `PrivacyInfo.xcprivacy` is present in `ios/`, referenced via `resource_bundles` in the podspec, and appears in Xcode Privacy Report with `NSPrivacyAccessedAPICategoryFileTimestamp` (C617.1) and `NSPrivacyAccessedAPICategoryDiskSpace` (85F4.1) declared; three successive `scan()` calls show no instance count growth in Instruments Allocations
**Plans**: TBD
**UI hint**: yes

### Phase 3: Android Native (ML Kit + ActivityEventListener)
**Goal**: Users on a real Android device with Play Services can scan documents and receive stable `file://` URIs; `isSupported()` returns false on no-GMS; `model_unavailable` is distinct from `unsupported`; no leaked promises
**Depends on**: Phase 1
**Requirements**: AND-01, AND-02, AND-03, AND-04, AND-05, AND-06, AND-07, AND-08, AND-09, AND-10
**Success Criteria** (what must be TRUE):
  1. On a real Android device with Play Services, `scan({ outputFormats: ['jpeg', 'pdf'] })` returns `{ kind: 'success', images: [...], pdfUri: '...', pageCount: N }` with all URIs pointing to readable files; no `content://` URI ever appears in the JS result
  2. On an AOSP/no-GMS emulator, `isSupported()` returns `false` and `scan()` returns `{ kind: 'error', code: 'unsupported' }` with a message that includes the detected Play Services version and a docs URL
  3. On a device with GMS but offline (airplane mode, model not yet downloaded), `scan()` returns `{ kind: 'error', code: 'model_unavailable' }` — not `'unsupported'`
  4. Killing the host Activity via ADB mid-scan does not leave the JS promise hanging; the next `scan()` call after app resume rejects cleanly with `'unavailable'`; a 10-page scan on a slow emulator produces no `FileNotFoundException` in LogCat
  5. `consumer-rules.pro` is present and an `assembleRelease` build of the example app does not throw `ClassNotFoundException` for `SystemScannerModule`, `SystemScannerPackage`, or `FileCopyHelper`
**Plans**: TBD

### Phase 4: Expo Config Plugin
**Goal**: An Expo consumer can add `react-native-system-scanner` to their `app.json` plugins array and get correct `NSCameraUsageDescription`, ML Kit Gradle dep, and optional install-time model bundling without touching any native files
**Depends on**: Phase 1
**Requirements**: PLUG-01, PLUG-02, PLUG-03, PLUG-04
**Success Criteria** (what must be TRUE):
  1. Adding `["react-native-system-scanner", { "cameraPermission": "Allow scanning documents" }]` to `app.json` and running `expo prebuild` writes `NSCameraUsageDescription` to `ios/[AppName]/Info.plist` with the configured string
  2. After `expo prebuild`, the example app's `android/app/build.gradle` contains `implementation("com.google.android.gms:play-services-mlkit-document-scanner:16.0.0")` and the app builds with `assembleDebug`
  3. With `"mlKitModelInstallTime": true`, `expo prebuild` adds `<meta-data android:name="com.google.mlkit.vision.DEPENDENCIES" android:value="docui"/>` to `AndroidManifest.xml`
  4. The README contains a "Manual Setup" section documenting the `Info.plist` key and `build.gradle` changes required for non-Expo consumers
**Plans**: TBD

### Phase 5: Example App + Test Matrix
**Goal**: The `example/` app demonstrates every documented code path — all three output formats, cancellation, and the unsupported-device error path — and a fully checked `TESTING.md` documents that all 27 test cases pass on real hardware
**Depends on**: Phase 2, Phase 3, Phase 4
**Requirements**: EX-01, EX-02, EX-03, EX-04, EX-05, EX-06, EX-07, EX-08
**Success Criteria** (what must be TRUE):
  1. The example app launches on a real iOS device, a real Android device, and an AOSP/no-GMS Android emulator without crashing; `isSupported()` is called on launch and the scan button is visually disabled when it returns false
  2. Tapping "Scan (image only)", "Scan (PDF only)", and "Scan (both)" each produce the expected result type on iOS and Android; scanned images render inline and PDF page count is displayed
  3. Backing out of the scanner modal shows a non-error "Cancelled" message in the UI with no red error screen
  4. On a no-GMS emulator or iOS Simulator, the unsupported state is shown to the user — the typed `code` and `message` from the error result are visible
  5. `TESTING.md` exists with 27 checkbox rows (3 output formats x {iOS, Android, Android-no-GMS} x {success, cancel, permission-denied}) and every checkbox is checked, confirming real-device verification
**Plans**: TBD
**UI hint**: yes

### Phase 6: CI + Docs + Ship
**Goal**: Green CI on all four jobs (TSC, Biome, iOS build, Android build); README earns under 60 seconds comprehension by a new visitor; `v0.1.0` is published to npm with no source maps; the library is discoverable in react-native-directory and Expo's third-party module list
**Depends on**: Phase 5
**Requirements**: CI-01, CI-02, CI-03, CI-04, CI-05, CI-06, REL-01, REL-02, REL-03, REL-04, REL-05, REL-06, REL-07, REL-08
**Success Criteria** (what must be TRUE):
  1. Opening a PR triggers a GitHub Actions run that completes all four jobs (TSC, Biome, iOS `pod install + xcodebuild`, Android `assembleDebug`) with green checkmarks; the iOS job uses `maxim-lobanov/setup-xcode@v1` with a pinned Xcode version
  2. A new visitor can find the install command, a 5-line usage example, supported platforms/versions, and the demo gif within 60 seconds of loading the README
  3. `npm pack --dry-run` produces a tarball containing only the declared files (`lib/`, `ios/`, `android/`, `plugin/`, `src/`, `*.podspec`, `README.md`, `CHANGELOG.md`, `LICENSE`) — no `*.map` source map files appear
  4. `react-native-system-scanner@0.1.0` is published to npm with `react-native >=0.82.0` as the declared peer dep floor; the GitHub release `v0.1.0` is tagged with the demo gif attached
  5. The library appears in react-native-directory with `newArchitecture: true` and the Expo third-party module list entry links to the correct npm page
**Plans**: TBD

## Progress

**Execution Order:**
Phases 2, 3, and 4 are independent after Phase 1 — a solo engineer works them sequentially (2 → 3 → 4). Phase 5 requires all three. Phase 6 requires all five.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Scaffold + Types + JS Skeleton | 0/TBD | Not started | - |
| 2. iOS Native (VisionKit + PDFKit) | 0/TBD | Not started | - |
| 3. Android Native (ML Kit + ActivityEventListener) | 0/TBD | Not started | - |
| 4. Expo Config Plugin | 0/TBD | Not started | - |
| 5. Example App + Test Matrix | 0/TBD | Not started | - |
| 6. CI + Docs + Ship | 0/TBD | Not started | - |

---
*Roadmap created: 2026-05-04*
*Last updated: 2026-05-04 after initial roadmap creation*
