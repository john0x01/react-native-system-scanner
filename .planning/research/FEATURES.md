# Feature Research

**Domain:** React Native library — system-provided modal document scanner UI (VisionKit + ML Kit)
**Researched:** 2026-05-04
**Confidence:** HIGH (platform APIs verified against official docs; competitor APIs directly inspected)

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing = product feels incomplete or untrustworthy.

| Feature | Why Expected | Complexity | JS vs Native | Notes |
|---------|--------------|------------|--------------|-------|
| `scan(options)` → discriminated-union result | Every picker/scanner library in the RN ecosystem returns a result (expo-image-picker, rn-document-scanner-plugin). No consumer should need to try/catch a normal cancel. | LOW (JS) / MEDIUM (native) | Both | JS wrapper normalises platform divergence; native does the actual scanning. Cancellation MUST be `{ kind: 'cancelled' }`, not a thrown error — rn-document-scanner-plugin uses a `status: 'cancel'` string on the result, which is better than throwing but still not type-narrowable. |
| Output format choice: image, PDF, or both | Document apps need PDFs for sharing; some apps need images for further processing. rn-document-scanner-plugin returns only JPEG file paths — consumers frequently open GitHub issues asking for PDF support. | LOW (option) / MEDIUM (iOS PDF composition) | Both | `outputFormats: ('jpeg' \| 'pdf')[]` or an `OutputFormat` literal union. iOS: PDFKit composition; Android: native PDF from `GmsDocumentScanningResult.pdf`. |
| `isSupported()` — platform capability check | Apps must gate the "Scan Document" button. Without this, UI crashes on unsupported devices (Mac Catalyst, simulators, Huawei no-GMS phones). rn-document-scanner-plugin has no `isSupported()` at all — known pain point. | LOW | JS + native | Returns `boolean`. Must check `VNDocumentCameraViewController.isSupported` on iOS; checks Play Services version >= required on Android. Does NOT mean camera is granted — document this explicitly. |
| Stable `file://` URIs that outlive the scanner session | Android ML Kit returns `content://` URIs that expire when the scanner Activity is destroyed. Consumers who store URIs and try to upload later get silent failures. expo-image-picker solves this by copying to its own cache directory — we must do the same. | MEDIUM | Native (both platforms) | Native copies to `<libCacheDir>/scans/<scanId>/` before resolving. JS generates the `scanId` (ULID via `ulidx`) and passes it to native. Dependency: JS-side ULID generation → native file path. |
| `cleanup(scanId?)` — delete library-managed files | Consumers must be able to free storage after upload/processing. Without this, the library is a storage leak. | LOW | JS + native | `cleanup()` with no args deletes all `<libCacheDir>/scans/`. `cleanup(scanId)` deletes one scan folder. Dependency: stable URI strategy must exist first. |
| Typed errors with distinct codes | `'unsupported'`, `'permission_denied'`, `'unavailable'`, `'unknown'` — consumers need to branch on error type for UX (show "update Play Services" vs "grant camera" vs "not available on this device"). rn-document-scanner-plugin throws untyped errors. | LOW | JS (error mapping layer) | Error codes as literal union, not enum. Message must include actionable detail: version found, URL to docs. Native surfaces errors; JS normalises to typed codes. |
| TypeScript types for everything (zero `any`) | Modern RN ecosystem expectation. MMKV, Reanimated, op-sqlite all ship zero-`any` strict TypeScript. Libraries without types are increasingly rejected. | LOW | JS | All exported symbols carry TSDoc. Public types use `type` (literal unions for discriminants); module shapes use `interface` (codegen requirement). |
| `maxPages` option | Limiting scan length is a product requirement (e.g., cap at 10 pages for a contract scanner). rn-document-scanner-plugin has `maxNumDocuments` (Android only). ML Kit has `setPageLimit()` (clamps ~25 internally). | LOW | Both | Pass through to ML Kit; iOS VisionKit has no page limit API — document that VisionKit ignores this option. Cap: 25 on Android; unlimited (user-controlled) on iOS. |
| Expo config plugin: iOS `NSCameraUsageDescription` | Without the camera permission string the app crashes on submit to the App Store and at runtime on iOS. Every camera library (expo-image-picker, expo-camera) ships this as a config plugin prop. | LOW | Build-time only | `withInfoPlist` in config plugin. Accepts `cameraPermission` string with a default fallback. Consumers who don't use Expo do manual `Info.plist` editing — document both paths in README. |
| Expo config plugin: Android ML Kit Gradle dependency | Without the `play-services-mlkit-document-scanner` Gradle dep added, the app builds but crashes at `scan()` call. Consumers expect one-line Expo plugin setup. | LOW | Build-time only | `withAppBuildGradle` adds `implementation("com.google.android.gms:play-services-mlkit-document-scanner:16.0.0")`. Dependency: must exist before Android scanner works at all. |

---

### Differentiators (Competitive Advantage)

Features that distinguish this library from the existing OpenCV-based alternatives.

| Feature | Value Proposition | Complexity | JS vs Native | Notes |
|---------|-------------------|------------|--------------|-------|
| Pure system UI — zero OpenCV/third-party camera dep | rn-document-scanner-plugin and react-native-rectangle-scanner bundle OpenCV (adds ~30 MB to APK) and ship their own camera views. This library ships zero ML code — ML Kit lives in Play Services (adds ~300 KB to app). iOS VisionKit is a system framework with zero size impact. | LOW (no work to add; just don't include) | N/A | The differentiator is what we DON'T ship. Document the size comparison explicitly in README. |
| Android scanner mode: `SCANNER_MODE_FULL` (default) vs `SCANNER_MODE_BASE` | `SCANNER_MODE_FULL` includes ML image cleaning (erase fingers, shadows, stains) — a UX capability OpenCV-based libs can't match. `SCANNER_MODE_BASE` is for apps that want faster, lighter scanning without ML. No existing RN library exposes this. | LOW (option pass-through) | JS (option) + native | `scannerMode: 'full' \| 'base'` option on `scan()`. Maps to ML Kit `SCANNER_MODE_FULL` / `SCANNER_MODE_BASE`. iOS ignores this option (no equivalent). Document the UX difference. Android only; default `'full'`. |
| Android gallery import toggle | `setGalleryImportAllowed` lets apps that want live-capture-only disable gallery import, or let document management apps enable it. No existing RN library exposes this flag. | LOW (option pass-through) | JS (option) + native (Android only) | `galleryImportAllowed: boolean` — Android only. iOS VisionKit always shows gallery import; no API to disable. Default: `true` (more capable default). Document iOS limitation. |
| Discriminated-union result (`kind: 'success' \| 'cancelled' \| 'error'`) | rn-document-scanner-plugin uses `status: 'success' \| 'cancel'` on a flat object — not type-narrowable without manual checking. This library's `kind` discriminant allows TypeScript `switch`/`if`-narrowing out of the box, following the pattern established by expo-image-picker's `canceled: boolean` but more expressive. | LOW | JS | The `kind` field on every result variant enables consumers to `switch (result.kind)` with exhaustive narrowing. Error is a result, not a thrown exception. |
| Expo config plugin: install-time vs on-demand ML Kit model | ML Kit Document Scanner models live in Play Services (on-demand by default). Apps that want zero first-run latency can bundle the model at install time via `<meta-data>` in AndroidManifest. No existing RN library exposes this choice. | LOW (config plugin mod) | Build-time only | `mlKitModelInstallTime: true` flag in config plugin. Adds `withAndroidManifest` modifier setting `com.google.mlkit.vision.DEPENDENCIES`. Default: `false` (smaller install, model downloads on first scan). |
| Typed, actionable error messages | Errors include the version found, the version required, and a URL to the relevant docs page. E.g., `"Play Services 21+ required. Detected: 19.8.39. See: https://..."`. No existing RN library does this — they throw opaque native exceptions. | LOW (string formatting) | Native + JS error mapping | Implemented in native (has version info) then surfaced via typed `code` + `message` fields on the error result. |

---

### Anti-Features (Deliberately Not Built)

| Anti-Feature | Why Requested | Why Not Building It | Alternative / What We Do Instead |
|--------------|---------------|--------------------|------------------------------------|
| Custom camera UI / fallback scanner | OpenCV-based libs have this; some users want it for unsupported devices | Contradicts the library's entire premise (system UI fidelity) and would balloon binary size by ~30 MB. Scope creep kills a 2-3 week ship target. | Return `{ kind: 'error', code: 'unsupported' }` with an actionable message. Document device requirements clearly. |
| OCR / text extraction | Logical next step after scanning | Completely different API surface (`Vision` framework on iOS, ML Kit Text Recognition on Android). Separate library scope. | Consumers can run the returned image URIs through `react-native-mlkit-ocr` or Apple's `VNRecognizeTextRequest` separately. |
| Barcode / QR scanning | Cameras → barcode is a common combo | Owned by `react-native-vision-camera` and `expo-barcode-scanner`. Different problem domain. | Use `react-native-vision-camera` + barcode plugin. |
| Post-scan editing (crop, rotate, filter) | Consumers want to refine output | Both system UIs (VisionKit and ML Kit) include their own editing UI. Duplicating this is a worse UX and wasted work. | System UIs handle editing in-flow. Users crop and filter before the result returns. |
| View component / embedded camera surface | Some consumers want an inline scanner | Both platform APIs are exclusively modal. There is nothing to embed. A view component would be an empty shell. | The imperative `scan()` call is the only correct API shape. |
| Old-architecture (bridge) support | Maximum compatibility | Doubles the codebase with legacy shim. Contradicts the TurboModule learning goal. RN 0.82+ makes new-arch mandatory. | Declare `>=0.82` peer floor and document explicitly in README. |
| iPad-specific presentation tweaks | iPad UX for split view, popover, etc. | `VNDocumentCameraViewController` defaults to fullscreen modal on iPad — acceptable UX, zero work. ML Kit is fullscreen modal by nature. Customising past OS defaults adds complexity for minimal gain. | Verify split-view works on real hardware (test matrix item). Don't customise. |
| JPEG quality option | Fine-grained image quality control | 0.85 is the well-established default for scanned documents (matches iOS Notes, ML Kit default). Feature adds API surface for negligible real-world gain. | Hard-code 0.85 in v0.1. Revisit in v0.2 only if filed as a genuine issue. |
| Base64 image output | rn-document-scanner-plugin supports it; some consumers use it | Encodes large images into memory as strings — causes OOM for multi-page documents. File URIs are the correct approach for large binary data. | Return `file://` URIs only. Consumers who need base64 can read the file with `expo-file-system` or `react-native-fs`. |
| `isSupported()` → camera permission status | Consumers sometimes ask if `isSupported` means "everything will work" | Camera permission is a runtime check that happens when `scan()` is called, not at query time. Mixing capability and permission in one call creates confusing semantics. | `isSupported()` answers "does this device have the hardware and OS version?". Camera denial is a distinct error code `'permission_denied'` on the scan result. |

---

## Feature Dependencies

```
scan() options + result shape (JS types)
    └──requires──> Stable URI strategy (native copy to <libCacheDir>/scans/<scanId>/)
                       └──requires──> ULID generation (JS, via ulidx)
                       └──enables──>  cleanup(scanId?) function

scan() PDF output
    └──requires──> Output format option (`outputFormats` array)
    └──requires (iOS)──> PDFKit composition (native iOS)
    └──requires (Android)──> RESULT_FORMAT_PDF passed to GmsDocumentScannerOptions

scan() Android scanner mode
    └──requires──> scannerMode option on scan()
    └──maps to──>  setScannerMode() on GmsDocumentScannerOptions

isSupported()
    └──enables (not requires)──> scan()
    └──does NOT guarantee──> camera permission (separate error code)

Expo config plugin: cameraPermission
    └──requires (Expo consumers only)──> expo peer dep >=56.0.0
    └──enables──> NSCameraUsageDescription in Info.plist

Expo config plugin: Gradle dependency
    └──requires (Expo consumers only)──> expo peer dep >=56.0.0
    └──enables──> play-services-mlkit-document-scanner in build.gradle

Expo config plugin: mlKitModelInstallTime
    └──requires──> Expo config plugin (Gradle dep) already wired
    └──enables──> install-time model bundling via AndroidManifest meta-data
```

### Dependency Notes

- **Stable URI requires ULID:** The `scanId` is generated in JS before the `scan()` native call; native writes files to `<libCacheDir>/scans/<scanId>/`. This means the ULID dependency (`ulidx`) is a hard dependency for the feature to work at all, not optional.
- **cleanup() requires stable URI:** `cleanup(scanId)` can only work if the library owns the file paths. If we ever changed the URI strategy, cleanup would break.
- **PDF output requires output format option:** The `outputFormats` field is the gate for whether iOS does PDFKit composition (expensive, memory-intensive) or skips it. Requesting both formats means native does both code paths.
- **Android scanner mode is Android-only:** iOS VisionKit has zero configuration knobs — Apple controls the entire UX. Any iOS-specific option on `scan()` has no effect and should be documented as no-op.
- **Config plugin is Expo-only:** Non-Expo consumers must wire `NSCameraUsageDescription` and Gradle manually. README must cover both paths.

---

## `scan()` Options — Concrete Shape

Grounded in platform API capabilities verified against official docs.

```typescript
type OutputFormat = 'jpeg' | 'pdf';
type ScannerMode = 'full' | 'base'; // Android only; iOS ignores

interface ScanOptions {
  /**
   * Which output formats to generate.
   * - 'jpeg': returns `images` (array of file:// URIs, one per page)
   * - 'pdf':  returns `pdfUri` (file:// URI to a multi-page PDF)
   * Both can be requested simultaneously; native generates both.
   * Default: ['jpeg']
   */
  outputFormats?: OutputFormat[];

  /**
   * Maximum number of pages the user can scan.
   * Android: passed to ML Kit setPageLimit() — ML Kit internally clamps at ~25.
   * iOS: VisionKit has no page limit API; this option is ignored on iOS.
   * Default: undefined (no limit; ML Kit defaults to its internal max ~25).
   */
  maxPages?: number;

  /**
   * Android only. Scanner feature set.
   * 'full': ML image cleaning (erase fingers, stains, shadows) + filters + basic.
   *         Enables future ML Kit capability upgrades automatically.
   * 'base': Basic editing only (crop, rotate, reorder, save). No ML cleaning.
   * iOS: ignored.
   * Default: 'full'
   */
  scannerMode?: ScannerMode;

  /**
   * Android only. Whether the user can import from the photo gallery inside the scanner UI.
   * iOS: VisionKit always shows gallery import; this option is ignored on iOS.
   * Default: true
   */
  galleryImportAllowed?: boolean;
}
```

**Rationale for each option:**
- `outputFormats`: Core feature; no existing RN lib exposes both. Grounded in `RESULT_FORMAT_JPEG` + `RESULT_FORMAT_PDF` from ML Kit and PDFKit on iOS.
- `maxPages`: Grounded in `setPageLimit()` on ML Kit; common consumer request in existing libs.
- `scannerMode`: Grounded in `SCANNER_MODE_FULL` / `SCANNER_MODE_BASE` from ML Kit; a genuine UX differentiator.
- `galleryImportAllowed`: Grounded in `setGalleryImportAllowed()` from ML Kit; low cost, exposes a real platform knob.

**NOT included in v0.1:**
- `quality` / `croppedImageQuality` — Hard-coded at 0.85; revisit v0.2 only if requested.
- Any iOS-specific options — VisionKit has zero configuration API.
- `captureMode` (`CAPTURE_MODE_AUTO` vs `CAPTURE_MODE_MANUAL`) — ML Kit exposes this but it duplicates `scannerMode`'s intent. `SCANNER_MODE_FULL` already enables auto-capture. Adding a separate capture mode option adds surface for minimal gain. Revisit in v0.2.

---

## Result Shape — Concrete and Validated

```typescript
type ScanResult =
  | ScanSuccessResult
  | ScanCancelledResult
  | ScanErrorResult;

interface ScanSuccessResult {
  kind: 'success';
  /**
   * File URIs (file://) for each scanned page as JPEG.
   * Present when 'jpeg' was in outputFormats.
   * Undefined when only 'pdf' was requested.
   */
  images?: string[];
  /**
   * File URI (file://) to the composed multi-page PDF.
   * Present when 'pdf' was in outputFormats.
   * Undefined when only 'jpeg' was requested.
   */
  pdfUri?: string;
  /**
   * Number of pages scanned. Always present on success.
   * Useful when caller requested only pdfUri and doesn't have the images array.
   */
  pageCount: number;
}

interface ScanCancelledResult {
  kind: 'cancelled';
}

type ScanErrorCode =
  | 'unsupported'      // VNDocumentCameraViewController.isSupported=false, or GMS too old / absent
  | 'permission_denied'// Camera permission denied at scan() time
  | 'unavailable'      // Scanner activity failed to launch or crashed mid-scan
  | 'unknown';         // Catch-all for unclassified native errors

interface ScanErrorResult {
  kind: 'error';
  code: ScanErrorCode;
  /**
   * Actionable message. Examples:
   * - "Play Services 21+ required. Detected: 19.8. See: https://..."
   * - "Camera permission denied. Request permission before calling scan()."
   * - "VNDocumentCameraViewController is not supported on this device."
   */
  message: string;
}
```

**Validation of the proposed shape from the question:**

The proposed shape `{ kind: 'success', images?: string[], pdfUri?: string, pageCount: number }` is correct. Two refinements:

1. `images` and `pdfUri` should be optional (not always-present), conditional on which `outputFormats` were requested. A caller who asked for only `pdf` should not receive an empty `images: []`.
2. `pageCount` is valuable even when only `pdfUri` is returned — it lets the UI show "Scanned 5 pages" without counting an array. Android provides this via `GmsDocumentScanningResult.pdf?.pageCount`; iOS computes it from `scan.pageCount`.

**Why `kind` not `status` (rn-document-scanner-plugin's choice):**
- `kind` is the established discriminant idiom in TypeScript (`{ kind: 'A' } | { kind: 'B' }`)
- `status: 'cancel'` in rn-document-scanner-plugin requires string comparison; `kind: 'cancelled'` enables TypeScript narrowing with `switch (result.kind)` → exhaustive checks.

**Why no thrown exceptions:**
- Cancellation is a normal user action, not an error. Throwing forces consumers into try/catch for expected control flow.
- Only genuine unrecoverable failures surface as `kind: 'error'`.

---

## MVP Definition

### v0.1 — Ship With

- [x] `scan(options: ScanOptions): Promise<ScanResult>` — core function, all four option fields
- [x] `isSupported(): Promise<boolean>` — capability check (not permission check)
- [x] `cleanup(scanId?: string): Promise<void>` — storage management
- [x] Stable `file://` URIs via ULID-keyed lib cache directory (both platforms)
- [x] Output formats: `jpeg`, `pdf`, or both
- [x] Discriminated-union result with typed error codes
- [x] Android `scannerMode` (`full` | `base`)
- [x] Android `galleryImportAllowed`
- [x] `maxPages` (Android respected; iOS documented as no-op)
- [x] Expo config plugin: `cameraPermission`, Gradle dependency, `mlKitModelInstallTime` flag
- [x] TypeScript strict, zero `any`, TSDoc on all exports
- [x] Example app demonstrating all 3 output modes + cancellation + error (unsupported device)

### Add After v0.1 Validation (v0.2 candidates)

- JPEG quality option (`quality: number`) — only if consumer requests file it as an issue
- Android `captureMode` (`auto` | `manual`) — separate from `scannerMode`; low demand currently
- CoreGraphics-based incremental PDF composition on iOS — only if OOM is measured on real devices with >10-page scans
- `TS 6.0` upgrade — once RN codegen confirms compatibility

### Future / v2+

- Expo SDK integration as an Expo module (requires `expo-modules-core` — different architecture)
- Multi-instance support (multiple concurrent `scan()` calls)
- Background upload of scan results (out of scope for a scanner library)

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| `scan()` with discriminated-union result | HIGH | MEDIUM | P1 |
| Stable `file://` URIs (copy-on-return) | HIGH | MEDIUM | P1 |
| Output format choice (jpeg/pdf/both) | HIGH | MEDIUM (iOS PDF composition) | P1 |
| `isSupported()` | HIGH | LOW | P1 |
| Typed error codes | HIGH | LOW | P1 |
| TypeScript types, TSDoc | HIGH | LOW | P1 |
| `cleanup()` | MEDIUM | LOW | P1 |
| `maxPages` option | MEDIUM | LOW | P1 |
| Android `scannerMode` | MEDIUM | LOW | P1 |
| Android `galleryImportAllowed` | LOW | LOW | P1 (low cost, high fidelity) |
| Expo config plugin (camera permission + Gradle) | HIGH | LOW | P1 |
| Expo config plugin ML Kit model timing flag | MEDIUM | LOW | P1 (same plugin, low marginal cost) |
| JPEG quality option | LOW | LOW | P3 |
| Android `captureMode` | LOW | LOW | P3 |
| Base64 output | LOW | LOW (but harmful) | Never |

---

## Competitor Feature Analysis

| Feature | rn-document-scanner-plugin (OpenCV) | react-native-rectangle-scanner (OpenCV) | react-native-system-scanner (this) |
|---------|-------------------------------------|------------------------------------------|--------------------------------------|
| Platform scanner API | No (custom OpenCV UI) | No (custom camera view) | Yes (VisionKit + ML Kit) |
| iOS PDF output | No | No | Yes (PDFKit) |
| Android PDF output | No | No | Yes (ML Kit native) |
| `isSupported()` check | No | No | Yes |
| Cancellation as non-error | Partial (`status: 'cancel'` but on same object) | No (callback-based) | Yes (`kind: 'cancelled'`) |
| Typed error codes | No (throws untyped) | No (callback error) | Yes (4 distinct codes) |
| Stable file URIs (lib-owned) | Unclear — returns temp paths | Returns temp paths | Yes (ULID-keyed, lib-owned) |
| `cleanup()` function | No | No | Yes |
| Android scanner mode | No | No | Yes (`full` / `base`) |
| Gallery import toggle | No | No | Yes (Android) |
| App binary size impact | ~30 MB (OpenCV) | ~30 MB (OpenCV) | ~300 KB (ML Kit via Play Services) |
| New architecture support | No (old-arch only) | No (old-arch only, archived) | Yes (TurboModule only) |
| Expo config plugin | No | No | Yes |
| TypeScript strict types | Partial | No | Yes |

---

## Sources

- react-native-document-scanner-plugin API — https://react-native-document-scanner.js.org/ (inspected May 2026)
- react-native-rectangle-scanner README — https://github.com/HarvestProfit/react-native-rectangle-scanner/blob/master/README.md (inspected May 2026, project archived)
- expo-image-picker result shape and type pattern — https://docs.expo.dev/versions/latest/sdk/imagepicker/ (verified May 2026)
- GmsDocumentScannerOptions API reference — https://developers.google.com/android/reference/com/google/mlkit/vision/documentscanner/GmsDocumentScannerOptions (verified May 2026)
- ML Kit Document Scanner Android guide — https://developers.google.com/ml-kit/vision/doc-scanner/android (updated Apr 22, 2026)
- ML Kit model installation paths — https://developers.google.com/ml-kit/tips/installation-paths (verified May 2026)
- Android Developers Blog on ML Kit Document Scanner — https://android-developers.googleblog.com/2024/02/ml-kit-document-scanner-api.html
- VNDocumentCameraViewController Apple Docs — https://developer.apple.com/documentation/visionkit/vndocumentcameraviewcontroller (verified; no configuration options exist)
- ML Kit in action (GDE article) — https://medium.com/google-developer-experts/ml-kit-document-scanner-in-action-1c3a49ef5a33

---

*Feature research for: react-native-system-scanner (TurboModule wrapping VisionKit + ML Kit)*
*Researched: 2026-05-04*
