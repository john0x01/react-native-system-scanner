<!-- GSD:project-start source:PROJECT.md -->
## Project

**react-native-system-scanner**

An open-source React Native library that lets RN apps scan documents using the **system-provided modal scanner UIs** on both platforms — Apple's `VNDocumentCameraViewController` (VisionKit) on iOS, and Google's `GmsDocumentScanning` (Play Services ML Kit) on Android. It is **not** a camera library; it sits at a different layer than `react-native-vision-camera` / `expo-camera` by wrapping the OS's modal scanner UIs rather than embedding camera views.

**Core Value:** A 5-line `scan()` call that opens the OS-native document scanner and returns a stable result — `success` with image and/or PDF URIs, `cancelled`, or `error` — without the consumer having to reason about cache eviction, permissions, or platform divergence.

### Constraints

- **Tech stack — Scaffold:** `create-react-native-library`, TurboModule template, no backward compat — new-arch only.
- **Tech stack — Native languages:** Swift on iOS, Kotlin on Android. No Obj-C, no Java.
- **Tech stack — Module shape:** TurboModule only; **no Fabric component**. The system UIs are modal — there is nothing to embed.
- **Tech stack — Bundling/tooling:** `react-native-builder-bob` (ESM + CJS), Biome for lint/format, pnpm for workspace ergonomics with `example/`.
- **Tech stack — TypeScript:** Strict mode, zero `any`. Public API uses literal unions (not string-literal enums). Types over interfaces for public unions; interfaces for module shapes (codegen requires).
- **Native style — Swift:** Prefer `async/await` where deployment target allows. No singletons unless Apple's API forces them.
- **Native style — Kotlin:** Coroutines for async work. No RxJava. No Java.
- **Platform support:** iOS 13+ (VisionKit). Android with recent Play Services (ML Kit Document Scanner, ~Sept 2023+).
- **Quality bar:** No `console.log` in shipped code. Every exported symbol has TSDoc. Error messages are actionable (include the missing version, link to docs). No silent failures — every native error path surfaces typed.
- **Timeline:** ~2-3 weeks to v0.1.0 ship. Small, sharp library; **not** a flagship. Scope discipline is critical.
- **License:** MIT.
- **Commit style:** Conventional commits, squash on PR merge. **Author-only — no AI co-author trailer.**
- **Public API surface (v0.1):** `scan()`, `isSupported()`, `cleanup()`, plus types. Adding to this surface requires explicit decision.
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Technologies
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| React Native | `>=0.82` (peer floor) | Target runtime | 0.82 (Oct 2025) was the first RN where new-arch is mandatory and can't be disabled; old bridge is dead. 0.85.x is current stable (Apr 2026). Setting `>=0.82` as the peer floor is honest — earlier versions require toggling new-arch on, adding friction. |
| React | `>=18.0` | Peer runtime | Follows RN peerDeps convention; 0.82+ ships with React 18. |
| TypeScript | `~5.8` or `~6.0` | Static typing | TS 5.8 (Mar 2025) is the last stable in the 5.x family. TS 6.0 (Mar 2026) ships strict-by-default, ESM-by-default, breaking changes. Pin `~5.8` in devDeps to avoid consuming TS 6 breakage until the ecosystem settles; set `tsconfig.json` to `"strict": true` explicitly. |
| Swift (iOS native) | Language only — no version pin | iOS native implementation | VisionKit and PDFKit are Swift-native frameworks. Swift async/await is available on iOS 15+; with iOS 13 deployment target, use callback-style delegation for VisionKit and Swift async for internal-only helpers. No Obj-C. |
| Kotlin (Android native) | Kotlin bundled with AGP / RN Gradle | Android native implementation | ML Kit Android SDK is Kotlin-first; coroutines are idiomatic for the async scanner launch flow. No Java. |
| VisionKit (iOS system framework) | iOS 13+ (framework ships with OS) | Document scanning UI on iOS | `VNDocumentCameraViewController` is the only OS-provided document scanner on iOS. No third-party dependency; zero binary size impact; Apple maintains it. |
| PDFKit (iOS system framework) | iOS 11+ (available, used for iOS 13+) | PDF composition on iOS | Converts `[UIImage]` → multi-page PDF. System framework, zero binary impact. No compression quality API exists — use `UIImage.jpegData(compressionQuality: 0.85)` before creating `PDFPage(image:)` to pre-compress. |
| Play Services ML Kit Document Scanner | `com.google.android.gms:play-services-mlkit-document-scanner:16.0.0` | Document scanning UI on Android | The only OS-integrated document scanner on Android. 16.0.0 is the current stable (Maven, 2024; docs updated Apr 2026). Scanner logic, UI, and models live in Play Services — minimal APK impact; on-demand model download by default. |
### Scaffolding & Build Tools
| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| `create-react-native-library` | `0.62.0` (Apr 2026 latest) | Scaffold library skeleton | Use `npx create-react-native-library@latest`. Template is `turbo-module` (not `turbo-module-mixed` — that adds old-arch shim you don't want). Language selection: choose "Kotlin & Objective-C" in the interactive prompt — Swift wiring must be done manually post-scaffold (see Swift Gotcha below). |
| `react-native-builder-bob` | `0.41.0` (Apr 2026 latest) | Build TS → ESM + CJS | Included automatically by `create-react-native-library`. Set `"esm": true` for ESM-only output (0.40+ default dropped dual CJS+ESM in favour of ESM-only; Metro ≥0.82 enables package exports). Node ≥20.19 required. |
| Biome | `@biomejs/biome@2.4.x` (2.4.14 latest May 2026) | Lint + format | Replaces ESLint + Prettier with a single Rust binary. ~10-20x faster. v2.0 added type-aware linting (no TS compiler needed), domains, and plugins. Use `biome.json` in root with `"linter": { "enabled": true }` and `"formatter": { "enabled": true }`. |
| pnpm | `>=9` | Package manager + workspace | Standard workspace ergonomics for `library/ + example/`. Add `pnpm-workspace.yaml` declaring both packages. Use `shamefully-hoist=true` in `.npmrc` so Metro can resolve hoisted RN deps. |
### Supporting Libraries
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `ulidx` | `2.4.1` | ULID generation | Used in the **JS layer** to generate the `<ulid>` path segment before calling `scan()`, passing it to the native side so native can write files to `<libCacheDir>/scans/<ulid>/`. Using JS-side generation keeps ULID as a pure coordination concern; native needs a stable path, not ULID semantics. The `ulid` package (original) is unmaintained. `ulidx` is TypeScript-native, ships ESM + CJS, uses `crypto.getRandomValues` — fits the library perfectly. No need for native JSI ULID; throughput is not a bottleneck here (one ULID per scan call). |
| `expo` (peer, optional) | `*` | Config plugin host | Declare as optional peerDependency in the config-plugin package (or workspace). The `expo` package itself provides the config plugin API — do NOT depend on `expo-modules-core` or `@expo/config-plugins` separately. Consumers who don't use Expo simply never install the peer. |
| `@expo/config-plugins` | Internal dep of `expo` | Config plugin types | Do NOT declare as a direct dep. The types come transitively via `expo`. |
### Development Tools (not shipped)
| Tool | Purpose | Notes |
|------|---------|-------|
| GitHub Actions | CI | TypeScript check (`tsc --noEmit`), Biome lint, JS tests, `pod install + xcodebuild`, `./gradlew assembleDebug` |
| `@react-native/babel-preset` | Babel config for example app | Comes with RN; don't add manually |
| Jest | JS unit tests | Scoped to pure-TS logic (result shape, error mapping); no native mocking needed for v0.1 |
## Validated Locked Decisions
### Scaffold: `create-react-native-library` + `turbo-module` template
# When prompted:
# - What type of library? → Turbo module (no backward compat)
# - Language?             → Kotlin & Objective-C
# - Workspace manager?    → pnpm
### Swift on iOS: Manually bridged, not scaffolded natively
### TypeScript: Strict, ~5.8
### React Native peer floor: `>=0.82`
- RN 0.74: new-arch available but opt-in (flag required).
- RN 0.76 (Oct 2024): new-arch default.
- RN 0.82 (Oct 2025): new-arch mandatory, bridge permanently removed.
- RN 0.85 (Apr 2026): current stable.
### react-native-builder-bob: ESM-only output (0.41.0)
### Biome: v2.x
### Android: ML Kit Document Scanner 16.0.0
| Option | Values | Notes |
|--------|--------|-------|
| `setScannerMode()` | `SCANNER_MODE_BASE`, `SCANNER_MODE_BASE_WITH_FILTER`, `SCANNER_MODE_FULL` | `FULL` (default) includes ML image cleaning (erase stains, fingers, etc.) and enables future major features automatically. `BASE` = crop/rotate/reorder/save only — no ML cleaning. Default to `FULL`. |
| `setResultFormats()` | `RESULT_FORMAT_JPEG`, `RESULT_FORMAT_PDF` | Can request both simultaneously. |
| `setPageLimit()` | `Int` (ML Kit internally clamps ~25) | Pass caller's value through; document the realistic cap (~25). |
| `setGalleryImportAllowed()` | `Boolean` | Whether user can import from photo library. |
| `GmsDocumentScanningResult` | `.pages` (`[DocumentPage]`, each with `.imageUri`), `.pdf?.uri`, `.pdf?.pageCount` | Android `content://` URIs — copy to lib-controlled storage immediately before resolving promise. |
### iOS: VisionKit `VNDocumentCameraViewController`
| Aspect | Detail |
|--------|--------|
| Framework | `VisionKit` (not `Vision`) — import `VisionKit` |
| iOS deployment target | iOS 13.0+ (`API_AVAILABLE(ios(13.0)) API_UNAVAILABLE(macos, tvos, watchos)`) |
| Mac Catalyst | Available from Catalyst 13.1, but `isSupported` returns `false` on Mac Catalyst, simulators, and some iPad configurations with camera restrictions |
| `isSupported` | Class property `VNDocumentCameraViewController.isSupported: Bool` — always check before presenting |
| Returns | `VNDocumentCameraScan` via delegate `documentCameraViewController(_:didFinishWith:)` |
| `VNDocumentCameraScan` API | `scan.pageCount: Int`, `scan.imageOfPage(at: Int) -> UIImage` — iterate 0..<scan.pageCount |
| Cancellation | `documentCameraViewControllerDidCancel(_:)` — not an error, must map to `{ kind: 'cancelled' }` |
| Error | `documentCameraViewController(_:didFailWithError:)` — map to `{ kind: 'error', ... }` |
| PDF output | VisionKit returns `UIImage` only; PDF composition is our job (PDFKit) |
| No capture mode option | Unlike ML Kit, VisionKit has no "base vs full" mode; Apple controls the UX |
### iOS: PDFKit for PDF composition
### Config plugin: `expo` peer only
### ULID generation: JS layer via `ulidx`
## Installation
# Scaffold (interactive)
# In library root — already installed by scaffold:
# react-native-builder-bob@0.41.0
# Add ULID (JS layer)
# Add Biome (devDep)
# Android: in library's build.gradle (not npm)
# implementation("com.google.android.gms:play-services-mlkit-document-scanner:16.0.0")
## Alternatives Considered
| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Scaffold | `create-react-native-library` | Nitro Modules template | Nitro is a different runtime (not TurboModule); doesn't serve the learning goal of TurboModule internals. Also experimental in CI as of Apr 2026. |
| Lint/format | Biome | ESLint + Prettier | ESLint requires separate Prettier integration, TypeScript ESLint plugin config, and a slower parse cycle. Biome is 10-20x faster, single config file, actively replacing both tools in 2026. |
| ULID package | `ulidx` | `ulid` (original) | `ulid` is unmaintained; outstanding compatibility issues. `ulidx` is TypeScript-native, ESM + CJS, maintained. |
| ULID generation site | JS layer | Native (Kotlin/Swift) | Negligible performance difference for one-per-call. Keeps native interfaces simpler; avoids a native C++ dep. |
| iOS PDF | PDFKit | CoreGraphics PDF context | PDFKit is higher-level and sufficient for v0.1. CoreGraphics is better for incremental large-document streaming; revisit in v0.2 if OOM is measured. |
| Android scanner | ML Kit `GmsDocumentScanning` | OpenCV + custom camera | OpenCV requires bundling a large native library (~30MB). ML Kit lives in Play Services — zero app size impact. The system UI is also objectively better UX. |
| TypeScript version | `~5.8` | `~6.0` | TS 6.0 has breaking module and strict changes. RN's codegen toolchain hasn't committed to 6.0 support yet. Pin 5.8 until ecosystem catches up. |
| RN peer floor | `>=0.82` | `>=0.74` | 0.74–0.81 require explicit new-arch opt-in; consumers might be on new-arch=off. `>=0.82` is cleaner because new-arch is mandatory there. |
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Old-arch compatible (`turbo-module-mixed`) template | Doubles the codebase with legacy bridge shim, contradicts the learning goal | `turbo-module` (new-arch only) template |
| Obj-C / `.m` files for iOS logic | Modern new-arch libraries (Reanimated, MMKV, op-sqlite) all use Swift. Obj-C is the generated bridge glue only | Swift for all real logic; thin `.mm` glue for codegen interop |
| Java on Android | Kotlin is the Android standard; coroutines for async are idiomatic | Kotlin |
| Fabric component | The system scanner UIs are modal; there is nothing to embed. A Fabric component would be an empty shell | TurboModule only |
| `expo-modules-core` as a peer dep for the config plugin | Expo docs (Apr 2026) say: depend on `expo`, not `expo-modules-core` separately | `expo: ">=56.0.0"` as optional peer |
| `ulid` (original npm package) | Unmaintained since ~2022; unresolved compatibility issues | `ulidx` |
| ESLint + Prettier | Two config files, slower, requires TS ESLint plugin wiring on greenfield | Biome 2.x |
| `PDFKit` without pre-compressing images | PDFKit has no JPEG quality API; default lossless encoding produces very large files | `UIImage.jpegData(compressionQuality: 0.85)` before `PDFPage(image:)` |
| Direct Kotlin `ActivityResultLauncher` registration in module init | Must be registered during Activity lifecycle, not lazily — will crash at runtime | `RegisterActivityContracts` hook pattern (expo-image-picker reference) |
## Version Compatibility
| Package | Compatible With | Notes |
|---------|-----------------|-------|
| `react-native-builder-bob@0.41` | Node ≥20.19 | Node 18 dropped. Ensure CI uses Node 20+ |
| `@biomejs/biome@2.4` | Node ≥18 | No RN-specific constraint |
| `create-react-native-library@0.62` | React Native 0.85 (scaffolded example) | Bumped to RN 0.85 in CRNL v0.61 |
| `ulidx@2.4.1` | ESM + CJS; Node ≥14 | Works in RN Metro bundler; `crypto.getRandomValues` available on RN runtime |
| `play-services-mlkit-document-scanner:16.0.0` | minSdkVersion ≥21, RAM ≥1.7GB, GMS required | Stable release; beta1 was Feb 2024, stable followed later in 2024 |
| `VNDocumentCameraViewController` | iOS 13.0+; Mac Catalyst 13.1 (isSupported=false) | Check `isSupported` before presenting; false on simulator + Mac |
| TypeScript `~5.8` | React Native codegen | TS 6.0 compatibility with RN codegen not yet confirmed; hold at 5.8 |
## Swift Gotcha: Codegen + Swift Bridge Detail
#import "RNSystemScannerModule.h"
#import "react_native_system_scanner-Swift.h"  // auto-generated by Xcode
- (void)scan:(JS::NativeSystemScanner::ScanOptions &)options
## Sources
- React Native releases/versions — https://reactnative.dev/versions (verified May 2026)
- React Native 0.82 blog post — https://reactnative.dev/blog/2025/10/08/react-native-0.82
- React Native 0.85 blog post — https://reactnative.dev/blog/2026/04/07/react-native-0.85
- create-react-native-library GitHub releases — https://github.com/callstack/react-native-builder-bob/releases (v0.62.0 Apr 2026)
- create-react-native-library CI template matrix — https://github.com/callstack/react-native-builder-bob/blob/main/.github/workflows/build-templates.yml (verified available templates)
- Swift support discussion (no official template yet) — https://github.com/callstack/react-native-builder-bob/discussions/425
- Swift in TurboModules guide — https://www.cristiangutu.pro/using-swift-in-fabric-components-and-turbo-modules/
- react-native-builder-bob@0.41.0 changelog — https://github.com/callstack/react-native-builder-bob/releases/tag/react-native-builder-bob@0.40.0
- Biome 2.4.x — https://biomejs.dev (latest 2.4.14 verified May 2026)
- Biome 2.0 features — https://biomejs.dev/blog/biome-v2-0-beta/
- TypeScript 5.8 — https://devblogs.microsoft.com/typescript/announcing-typescript-5-8/
- TypeScript 6.0 — https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/ (Mar 2026)
- ML Kit Document Scanner Android docs — https://developers.google.com/ml-kit/vision/doc-scanner/android (updated Apr 22, 2026)
- GmsDocumentScannerOptions API ref — https://developers.google.com/android/reference/com/google/mlkit/vision/documentscanner/GmsDocumentScannerOptions
- play-services-mlkit-document-scanner Maven — https://mvnrepository.com/artifact/com.google.android.gms/play-services-mlkit-document-scanner/16.0.0
- VNDocumentCameraViewController Apple docs — https://developer.apple.com/documentation/visionkit/vndocumentcameraviewcontroller
- VNDocumentCameraScan Apple docs — https://developer.apple.com/documentation/visionkit/vndocumentcamerascan
- PDFKit Apple docs — https://developer.apple.com/documentation/pdfkit
- PDFKit JPEG compression workaround — https://forums.swift.org/t/compressing-images-for-pdfkit/44679
- expo-image-picker ActivityResultLauncher pattern — https://github.com/expo/expo/blob/main/packages/expo-image-picker (source read)
- Expo config plugin peer dep guidance — https://docs.expo.dev/config-plugins/development-for-libraries/ (updated Apr 2, 2026)
- ulidx npm — https://www.npmjs.com/package/ulidx (v2.4.1)
- ulid original — https://www.npmjs.com/package/ulid (unmaintained)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
