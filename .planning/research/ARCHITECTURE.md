# Architecture Research

**Domain:** React Native TurboModule library — system-provided modal document scanner (VisionKit + ML Kit)
**Researched:** 2026-05-04
**Confidence:** HIGH (file structure from official RN docs + CRNL template inspection; data flow from platform API docs; ActivityResultLauncher pattern from expo-image-picker source + community patterns)

---

## Standard Architecture

### System Overview

```
┌────────────────────────────────────────────────────────────────────┐
│  Consumer App (JS/TS)                                              │
│    const result = await scan({ outputFormats: ['jpeg', 'pdf'] })   │
└───────────────────────┬────────────────────────────────────────────┘
                        │  import { scan, isSupported, cleanup }
┌───────────────────────▼────────────────────────────────────────────┐
│  JS Layer  (src/)                                                  │
│  ┌─────────────────────┐  ┌───────────────────┐  ┌─────────────┐  │
│  │  scan.ts            │  │  index.ts          │  │  types.ts   │  │
│  │  - ULID generation  │  │  - public exports  │  │  - unions   │  │
│  │  - opts normalize   │  │                   │  │  - options  │  │
│  │  - promise wrapper  │  │                   │  │  - errors   │  │
│  └──────────┬──────────┘  └───────────────────┘  └─────────────┘  │
│             │ calls                                                │
│  ┌──────────▼────────────────────────────────┐                     │
│  │  NativeSystemScanner.ts  (codegen spec)   │                     │
│  │  interface Spec extends TurboModule { }   │                     │
│  └──────────┬────────────────────────────────┘                     │
└─────────────┼──────────────────────────────────────────────────────┘
              │  JSI (TurboModule bridge — no serialization)
    ┌─────────┴───────────────┐
    │                         │
┌───▼────────────────┐  ┌─────▼──────────────────────────────────────┐
│  iOS Native (ios/) │  │  Android Native (android/)                 │
│                    │  │                                            │
│  .mm (thin glue)   │  │  SystemScannerModule.kt                    │
│    ↓               │  │    extends NativeSystemScannerSpec         │
│  SystemScanner     │  │    implements ActivityEventListener        │
│  Impl.swift        │  │    ↓                                       │
│    ↓               │  │  ScannerLauncher.kt                       │
│  VisionKit         │  │    ActivityResultLauncher                  │
│  VNDocumentCamera  │  │    registered in MainActivity              │
│  ViewController    │  │    ↓                                       │
│    ↓               │  │  ML Kit GmsDocumentScanning               │
│  PDFComposer.swift │  │    GmsDocumentScanningResult               │
│    ↓               │  │    ↓                                       │
│  NSCachesDirectory │  │  FileCopyHelper.kt                        │
│  /scans/<ulid>/    │  │    content:// → cacheDir/scans/<ulid>/    │
└────────────────────┘  └────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | File | Responsibility |
|-----------|------|---------------|
| Public API surface | `src/index.ts` | Re-exports `scan`, `isSupported`, `cleanup`, all types |
| JS scan wrapper | `src/scan.ts` | Generates ULID, normalises options, calls TurboModule, returns typed result |
| Codegen spec | `src/NativeSystemScanner.ts` | TypeScript spec; codegen derives Obj-C++ + Kotlin interfaces |
| Types | `src/types.ts` | `ScanOptions`, `ScanResult`, `ScanErrorCode` — zero `any`, literal unions |
| iOS glue | `ios/RNSystemScannerModule.mm` | ~20-line Obj-C++ bridge; delegates every call to Swift |
| iOS implementation | `ios/RNSystemScannerImpl.swift` | VisionKit presentation, delegate callbacks, PDFKit composition |
| iOS PDF composition | `ios/PDFComposer.swift` | `[UIImage]` → `PDFDocument`; isolated so it can be swapped for CoreGraphics |
| Android module | `android/.../SystemScannerModule.kt` | TurboModule impl; implements `ActivityEventListener`; stores pending promise |
| Android launcher | `android/.../ScannerLauncher.kt` | Wraps `ActivityResultLauncher` registration; registered during `onCreate` via `MainActivityDelegate` |
| Android file helper | `android/.../FileCopyHelper.kt` | Copies `content://` URI bytes to `cacheDir/scans/<ulid>/`; also handles GMS version check |
| Android package | `android/.../SystemScannerPackage.kt` | Registers `SystemScannerModule` with RN's module registry |
| Expo config plugin | `plugin/src/index.ts` | `withInfoPlist` (camera permission), `withAppBuildGradle` (Gradle dep), `withAndroidManifest` (model flag) |

---

## Recommended Project Structure

```
react-native-system-scanner/
│
├── src/                                # TypeScript layer
│   ├── index.ts                        # Public surface: re-exports only
│   ├── scan.ts                         # Scan wrapper: ULID + option normalize
│   ├── NativeSystemScanner.ts          # Codegen spec (Native<NAME>.ts convention)
│   └── types.ts                        # ScanOptions, ScanResult, ScanErrorCode
│
├── ios/                                # iOS native
│   ├── RNSystemScannerModule.h         # Obj-C header (required by codegen)
│   ├── RNSystemScannerModule.mm        # Thin Obj-C++ bridge — delegates to Swift
│   ├── RNSystemScannerImpl.swift       # Real implementation: VisionKit + PDFKit
│   ├── PDFComposer.swift               # [UIImage] → PDFDocument composition
│   └── react-native-system-scanner.podspec
│
├── android/
│   └── src/main/
│       ├── AndroidManifest.xml
│       └── java/com/systemscanner/
│           ├── SystemScannerModule.kt  # TurboModule; ActivityEventListener
│           ├── ScannerLauncher.kt      # ActivityResultLauncher registration helper
│           ├── FileCopyHelper.kt       # content:// → cache copy; GMS check
│           └── SystemScannerPackage.kt # Package registration
│
├── plugin/                             # Expo config plugin (separate workspace)
│   ├── src/
│   │   └── index.ts                    # withInfoPlist + withAppBuildGradle + withAndroidManifest
│   └── package.json                    # name: react-native-system-scanner/plugin
│
├── example/                            # pnpm workspace: example app
│   ├── src/
│   │   └── App.tsx                     # Demonstrates all 3 output formats + cancel + error
│   ├── android/
│   │   └── app/src/main/java/.../
│   │       └── MainActivity.kt         # Hosts ScannerLauncher registration
│   ├── ios/
│   └── package.json
│
├── package.json                        # Library root
├── react-native.config.js
├── tsconfig.json
├── biome.json
├── pnpm-workspace.yaml
└── .github/workflows/ci.yml
```

### Structure Rationale

- **`src/NativeSystemScanner.ts`:** Must follow the `Native<ModuleName>.ts` naming convention so codegen can discover it via the `codegenConfig` key in `package.json`. Anything else causes codegen to skip the file silently.
- **`ios/RNSystemScannerModule.mm` + `RNSystemScannerImpl.swift` split:** The `.mm` file is what codegen-generated protocols point to (required); Swift cannot conform to the generated Obj-C protocol directly. All logic goes in `.swift`. The split keeps the bridge thin and replaceable.
- **`ios/PDFComposer.swift` separate from `RNSystemScannerImpl.swift`:** PDF composition is the highest-complexity, highest-risk piece (memory spikes, page-count edge cases). Isolating it simplifies testing and makes the CoreGraphics swap (v0.2) a single-file replacement.
- **`android/ScannerLauncher.kt` separate from `SystemScannerModule.kt`:** The `ActivityResultLauncher` must be registered in the host Activity's `onCreate`, not inside the TurboModule. Keeping it in a separate class makes the wiring explicit and prevents "registered too late" crashes.
- **`plugin/` as separate workspace member:** Config plugins can be published as a sub-path export (`react-native-system-scanner/plugin`) or as a separate package. The sub-path export pattern (used by `expo-image-picker`) is cleanest — one package, no separate publish step. Declare `"plugin"` as a `exports` entry in `package.json`.
- **`example/android/.../MainActivity.kt`:** This is the only file in the example that has library-specific wiring (the `ScannerLauncher` registration). Document it prominently in README as the one manual step for non-Expo consumers.

---

## Architectural Patterns

### Pattern 1: Codegen Spec → TurboModule Bridge

**What:** Define the JS-Native interface in a single TypeScript file following codegen naming convention. Codegen generates native type-safe stubs; native implements the generated abstract class/protocol.

**When to use:** Always — this is the TurboModule contract. No manual type mapping between JS and native.

**Trade-offs:** Requires sticking to codegen-supported types (`string`, `boolean`, `number`, `Object`, `Array`, `Promise`); cannot pass complex Swift/Kotlin types directly. Works well for this library's surface.

```typescript
// src/NativeSystemScanner.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface NativeScanOptions {
  scanId: string;         // ULID generated in JS
  outputFormats: string[];
  maxPages?: number;
  scannerMode?: string;   // 'full' | 'base' — Android only
  galleryImportAllowed?: boolean;
}

export interface NativeScanResult {
  kind: string;           // 'success' | 'cancelled' | 'error'
  images?: string[];      // file:// URIs
  pdfUri?: string;        // file:// URI
  pageCount?: number;
  code?: string;          // error code
  message?: string;       // error message
}

export interface Spec extends TurboModule {
  scan(options: NativeScanOptions): Promise<NativeScanResult>;
  isSupported(): Promise<boolean>;
  cleanup(scanId?: string): Promise<void>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeSystemScanner');
```

### Pattern 2: JS Wrapper Owns ULID + Result Shaping

**What:** `scan.ts` is the public-facing function. It generates the ULID, normalizes options with defaults, calls the TurboModule, and reshapes the native result (flat object with string discriminant) into the typed discriminated union.

**When to use:** Anytime the native layer returns a weakly-typed representation that needs to be promoted to TypeScript's full discriminated union. Keeps native code simple (returns plain objects); keeps JS types rich.

**Trade-offs:** One extra JS call per scan (negligible). Means native has no ULID logic — it receives the ULID as a string argument.

```typescript
// src/scan.ts
import { ulid } from 'ulidx';
import NativeSystemScanner from './NativeSystemScanner';
import type { ScanOptions, ScanResult } from './types';

export async function scan(options: ScanOptions = {}): Promise<ScanResult> {
  const scanId = ulid();
  const raw = await NativeSystemScanner.scan({
    scanId,
    outputFormats: options.outputFormats ?? ['jpeg'],
    maxPages: options.maxPages,
    scannerMode: options.scannerMode ?? 'full',
    galleryImportAllowed: options.galleryImportAllowed ?? true,
  });

  // Promote raw native object to typed discriminated union
  if (raw.kind === 'success') {
    return { kind: 'success', images: raw.images, pdfUri: raw.pdfUri, pageCount: raw.pageCount! };
  }
  if (raw.kind === 'cancelled') {
    return { kind: 'cancelled' };
  }
  return { kind: 'error', code: raw.code as ScanErrorCode, message: raw.message! };
}
```

### Pattern 3: iOS Obj-C++ Glue + Swift Implementation

**What:** Codegen generates an Obj-C++ protocol. The `.mm` file conforms to it by delegating every method to a `@objc public class` in Swift. Swift holds all real logic.

**When to use:** Mandatory for Swift TurboModules — Swift cannot conform to the codegen-generated Obj-C++ protocol directly.

**Trade-offs:** Adds one indirection layer (~20 lines of boilerplate). Stable, production-proven (Expo team uses this pattern). Not scaffolded by `create-react-native-library` — manual post-scaffold step.

```objc
// ios/RNSystemScannerModule.mm  (generated; keep thin)
#import "RNSystemScannerModule.h"
#import "react_native_system_scanner-Swift.h"  // auto-generated by Xcode

@implementation RNSystemScannerModule

RCT_EXPORT_MODULE()

- (void)scan:(JS::NativeSystemScanner::NativeScanOptions &)options
      resolve:(RCTPromiseResolveBlock)resolve
       reject:(RCTPromiseRejectBlock)reject {
    [RNSystemScannerImpl scanWithOptions:options resolve:resolve reject:reject];
}

- (void)isSupported:(RCTPromiseResolveBlock)resolve
              reject:(RCTPromiseRejectBlock)reject {
    [RNSystemScannerImpl isSupportedWithResolve:resolve reject:reject];
}

- (void)cleanup:(NSString *)scanId
        resolve:(RCTPromiseResolveBlock)resolve
          reject:(RCTPromiseRejectBlock)reject {
    [RNSystemScannerImpl cleanupWithScanId:scanId resolve:resolve reject:reject];
}

@end
```

### Pattern 4: Android ActivityEventListener for Activity Results

**What:** For TurboModules that need to handle activity results (i.e., the scanner Activity finishing), implement `ActivityEventListener` and register with `reactContext.addActivityEventListener(this)`. This is the established pattern for React Native modules that don't use `expo-modules-core`.

**When to use:** Any TurboModule that must receive `onActivityResult` callbacks. The modern `registerForActivityResult` / `ActivityResultLauncher` API requires registration before the Activity starts — which is not guaranteed when a TurboModule is initialized. `ActivityEventListener` is the safe, established alternative.

**Trade-offs:** Uses deprecated `onActivityResult` under the hood (React Native's internal `BaseActivityEventListener`), but `BaseActivityEventListener.onActivityResult` is not itself deprecated in the RN API surface — it's the RN framework's abstraction. The `ActivityResultLauncher` approach requires registering in `MainActivity.onCreate`, which imposes a manual step on non-Expo consumers. For this library, `ActivityEventListener` keeps the module self-contained.

```kotlin
// android/src/main/java/com/systemscanner/SystemScannerModule.kt
class SystemScannerModule(
    private val reactContext: ReactApplicationContext
) : NativeSystemScannerSpec(reactContext), ActivityEventListener {

    private var pendingPromise: Promise? = null
    private val REQUEST_CODE = 7412  // arbitrary; unique per module

    init {
        reactContext.addActivityEventListener(this)
    }

    override fun scan(options: ReadableMap, promise: Promise) {
        val activity = currentActivity
            ?: return promise.reject("unavailable", "No current Activity", null)

        pendingPromise = promise
        val scanId = options.getString("scanId")!!
        // Build GmsDocumentScannerOptions from options, then:
        val scanner = GmsDocumentScanning.getClient(scannerOptions)
        scanner.getStartScanIntent(activity)
            .addOnSuccessListener { intentSender ->
                activity.startIntentSenderForResult(
                    intentSender, REQUEST_CODE, null, 0, 0, 0
                )
            }
            .addOnFailureListener { e ->
                pendingPromise = null
                promise.reject("unavailable", e.message, e)
            }
    }

    override fun onActivityResult(
        activity: Activity, requestCode: Int, resultCode: Int, data: Intent?
    ) {
        if (requestCode != REQUEST_CODE) return
        val promise = pendingPromise ?: return
        pendingPromise = null

        when (resultCode) {
            Activity.RESULT_OK -> {
                val result = GmsDocumentScanningResult.fromActivityResultIntent(data)
                // Copy content:// URIs to stable storage, then resolve
                FileCopyHelper.copyResultAsync(reactContext, result, scanId, promise)
            }
            Activity.RESULT_CANCELED -> promise.resolve(mapOf("kind" to "cancelled"))
            else -> promise.reject("unknown", "Scanner returned unexpected result code", null)
        }
    }

    override fun onNewIntent(intent: Intent?) {}  // required by ActivityEventListener
}
```

---

## Data Flow

### iOS: End-to-End `scan()` Call

```
JS: scan({ outputFormats: ['jpeg', 'pdf'], maxPages: 10 })
  │
  ├─ [JS] ulid() → scanId = "01HX..."
  ├─ [JS] normalize options: { scanId, outputFormats: ['jpeg','pdf'], maxPages: 10, ... }
  │
  ▼ JSI call (no serialization overhead)
NativeSystemScanner.scan({ scanId, outputFormats, maxPages, ... })
  │
  ▼ Obj-C++ bridge (RNSystemScannerModule.mm) — thin delegation
[RNSystemScannerImpl scanWithOptions:options resolve:resolve reject:reject]
  │
  ▼ Swift (RNSystemScannerImpl.swift)
1. Check VNDocumentCameraViewController.isSupported
   └─ false → resolve({ kind:'error', code:'unsupported', message:... })
2. Check camera permission (AVAuthorizationStatus)
   └─ denied → resolve({ kind:'error', code:'permission_denied', message:... })
3. Create VNDocumentCameraViewController
4. Set self as delegate (VNDocumentCameraViewControllerDelegate)
5. Present modal on rootViewController (main thread)
  │
  ▼ [User scans pages inside VisionKit modal UI]
  │
  ├─ Cancel path:
  │    documentCameraViewControllerDidCancel(_:)
  │    → dismiss modal
  │    → resolve({ kind: 'cancelled' })
  │
  ├─ Error path:
  │    documentCameraViewController(_:didFailWithError:)
  │    → dismiss modal
  │    → resolve({ kind: 'error', code: 'unavailable', message: error.localizedDescription })
  │
  └─ Success path:
       documentCameraViewController(_:didFinishWith scan:)
       → dismiss modal
       → imageOfPage(at: 0..N-1) → [UIImage]
       │
       ├─ outputFormats includes 'jpeg':
       │    for each UIImage:
       │      jpeg = image.jpegData(compressionQuality: 0.85)
       │      write to NSCachesDirectory/scans/<scanId>/page-<n>.jpg
       │
       └─ outputFormats includes 'pdf':
            PDFComposer.compose(images, to: NSCachesDirectory/scans/<scanId>/scan.pdf)
              → for each UIImage in autoreleasepool:
                  compressed = UIImage(data: jpegData(compressionQuality:0.85))
                  page = PDFPage(image: compressed)
                  pdfDocument.insert(page, at: index)
              → pdfDocument.write(to: outputURL)
       │
       → resolve({
           kind: 'success',
           images: ['file:///.../<scanId>/page-0.jpg', ...],
           pdfUri: 'file:///.../<scanId>/scan.pdf',
           pageCount: N
         })
  │
  ▼ Back in JS (scan.ts)
Promote raw native result → typed ScanResult discriminated union
Return to consumer
```

### Android: End-to-End `scan()` Call

```
JS: scan({ outputFormats: ['jpeg', 'pdf'], maxPages: 10 })
  │
  ├─ [JS] ulid() → scanId = "01HX..."
  ├─ [JS] normalize options
  │
  ▼ JSI call
NativeSystemScanner.scan({ scanId, outputFormats, maxPages, scannerMode:'full', galleryImportAllowed:true })
  │
  ▼ Kotlin (SystemScannerModule.kt)
1. currentActivity check → null → reject('unavailable')
2. GMS version check (FileCopyHelper.isGmsAvailable(reactContext))
   └─ too old / absent → resolve({ kind:'error', code:'unsupported', message:"Play Services 21+ required..." })
3. Build GmsDocumentScannerOptions:
     .setScannerMode(SCANNER_MODE_FULL)
     .setResultFormats(RESULT_FORMAT_JPEG, RESULT_FORMAT_PDF)  // based on outputFormats
     .setPageLimit(10)
     .setGalleryImportAllowed(true)
4. pendingPromise = promise
5. GmsDocumentScanning.getClient(options)
     .getStartScanIntent(activity)
     .addOnSuccessListener { intentSender →
         activity.startIntentSenderForResult(intentSender, REQUEST_CODE, ...)
       }
     .addOnFailureListener { e → pendingPromise=null; reject }
  │
  ▼ [ML Kit scanner Activity launches as modal]
  ▼ [User scans pages]
  │
  ▼ ActivityEventListener.onActivityResult(activity, REQUEST_CODE, resultCode, data)
  │
  ├─ RESULT_CANCELED:
  │    resolve({ kind: 'cancelled' })
  │
  ├─ RESULT_OK:
  │    result = GmsDocumentScanningResult.fromActivityResultIntent(data)
  │    │
  │    └─ FileCopyHelper.copyResultAsync(context, result, scanId, promise)
  │         *** CRITICAL: do this BEFORE the scanner Activity fully finishes ***
  │         for each page in result.pages:
  │           src = page.imageUri   (content:// — valid NOW, expires soon)
  │           dst = cacheDir/scans/<scanId>/page-<n>.jpg
  │           contentResolver.openInputStream(src).copyTo(FileOutputStream(dst))
  │         if pdf requested:
  │           src = result.pdf.uri  (content:// — same expiry constraint)
  │           dst = cacheDir/scans/<scanId>/scan.pdf
  │           copy bytes
  │         resolve({
  │           kind: 'success',
  │           images: ['file:///.../<scanId>/page-0.jpg', ...],
  │           pdfUri: 'file:///.../<scanId>/scan.pdf',
  │           pageCount: result.pdf?.pageCount ?? result.pages?.size
  │         })
  │
  └─ Other codes:
       resolve({ kind: 'error', code: 'unknown', message: "resultCode: $resultCode" })
  │
  ▼ Back in JS (scan.ts)
Promote raw native result → typed ScanResult discriminated union
Return to consumer
```

---

## Boundary Ownership Table

| Step | Owned By | What Crosses the Boundary |
|------|----------|--------------------------|
| ULID generation | JS (`scan.ts`) | `scanId: string` passes to native as part of options |
| Option normalization & defaults | JS (`scan.ts`) | Normalized options object |
| Type-safe native call | Codegen (TurboModule) | Typed method invocation via JSI |
| Camera permission check (iOS) | Swift (`RNSystemScannerImpl.swift`) | Surfaces as `code: 'permission_denied'` in result |
| Scanner presentation (iOS) | VisionKit (OS) | Modal UI, no cross-boundary data until user finishes |
| Delegate callbacks (iOS) | Swift (delegate methods) | `VNDocumentCameraScan`, `Error`, cancel signal |
| PDF composition (iOS) | `PDFComposer.swift` | Reads `[UIImage]` from delegate; writes file to disk |
| File writes (iOS) | `RNSystemScannerImpl.swift` | Writes JPEG/PDF to `NSCachesDirectory/scans/<scanId>/` |
| Scanner Activity launch (Android) | `SystemScannerModule.kt` | `intentSender` from ML Kit; `REQUEST_CODE` for matching |
| GMS version check (Android) | `FileCopyHelper.kt` | Surfaces as `code: 'unsupported'` in result |
| Activity result callback (Android) | `ActivityEventListener` → `SystemScannerModule` | `resultCode`, `Intent` (containing `content://` URIs) |
| content:// copy (Android) | `FileCopyHelper.kt` | Reads `content://`, writes to `cacheDir/scans/<scanId>/` |
| Result shaping | JS (`scan.ts`) | Raw native object → `ScanResult` discriminated union |

---

## Recommended Defaults for Open Decisions

### ULID Generation: JS-Side

**Decision:** Generate ULID in `scan.ts` before the TurboModule call.

**Rationale:** Performance is not a concern for one ID per user-initiated scan. JS-side generation keeps the ULID purely a coordination mechanism — native needs a stable directory name, not ULID semantics. Avoids a native dependency; keeps native interfaces simpler. Native receives `scanId` as a plain string argument.

### Cache Directory Location

**iOS:** `NSCachesDirectory` (via `FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask).first!`) + `/scans/<ulid>/`

**Android:** `context.cacheDir` (app-private, not world-readable) + `/scans/<ulid>/`

**Rationale:** Both `NSCachesDirectory` and `cacheDir` are:
- App-private (no external storage permissions needed)
- OS-managed (purged under storage pressure — acceptable, documented)
- Available without requesting any permissions
- Stable within a process lifetime (not `NSTemporaryDirectory` which can be purged mid-session)

**Why not `filesDir` or `Documents/`:** `filesDir` is for persistent user data — scan results are ephemeral working copies; consumers are responsible for moving them to permanent storage after upload. `Documents/` is accessible via Files app on iOS, which adds undesirable visibility.

**Document the OS-purge behaviour explicitly:** `cleanup()` should always be called after the consumer has finished processing scan results. Under storage pressure, the OS may purge the cache directory on its own — this is by design, not a bug.

### cleanup() Semantics

**Default (no args):** Delete entire `<cacheDir>/scans/` directory and recreate it empty.

**With `scanId` arg:** Delete only `<cacheDir>/scans/<scanId>/`.

**Recommendation:** Do NOT implement time-based expiry (N seconds old) in v0.1. Reasons:
1. Time-based logic requires storing metadata (when each scan was written) — unnecessary complexity.
2. The consumer knows when they're done with a scan; an API that takes explicit control (`cleanup(scanId)` after upload) is safer than implicit time-based eviction.
3. `cleanup()` with no args is a nuclear option for "wipe all scanner cache" — useful in logout flows.

**No auto-cleanup on next `scan()` call.** Each scan gets its own ULID directory; old ones persist until explicitly cleaned up. Document this clearly so consumers know they own the cleanup responsibility.

### Error Code Taxonomy

```typescript
type ScanErrorCode =
  | 'unsupported'        // VNDocumentCameraViewController.isSupported=false,
                         // or GMS absent / version too old. Check isSupported() first.
  | 'permission_denied'  // Camera access denied at scan() call time.
                         // isSupported()=true does NOT mean camera is granted.
  | 'unavailable'        // Scanner failed to launch (Activity unavailable,
                         // intentSender error from ML Kit, runtime crash).
  | 'model_unavailable'  // ML Kit model failed to download / initialise.
                         // Separate from 'unsupported' — device has GMS but
                         // model is temporarily unavailable (network issue, etc.)
  | 'cancelled_by_system'// OS terminated the scanner UI (phone call, system kill).
                         // Different from user cancel (which is { kind:'cancelled' }).
  | 'unknown';           // Catch-all for unclassified native exceptions.
                         // Always include message with original error detail.
```

**Note on `model_unavailable`:** ML Kit Document Scanner can fail at `getStartScanIntent()` if the model hasn't downloaded yet and the network is unavailable. This is distinct from `unsupported` (wrong GMS version). Surfacing this distinctly lets consumers show "Check your connection and try again" instead of "Not supported on this device."

**Note on `cancelled_by_system`:** Android's `onActivityResult` can return codes other than `RESULT_OK` (1) and `RESULT_CANCELED` (0) — e.g., `RESULT_FIRST_USER` (Activity was killed by system). Map anything that isn't explicitly 0 or 1 to `cancelled_by_system` rather than `unknown`.

### File URI Format: `file://` URIs (not bare paths)

**Decision:** All paths in the result are `file://` URIs, not absolute paths.

**Rationale:**
- Consistent with how React Native's `Image` component, `fetch()`, and `expo-file-system` expect media URIs.
- `file:///path/to/file.jpg` can be used directly in `<Image source={{ uri }} />` without transformation.
- Bare paths (`/var/mobile/...`) require consumers to prepend `file://` manually, which is error-prone.
- Android already returns `content://` from ML Kit (which we copy to `file://`); `file://` is the stable format.

**Implementation:** Native layer writes files and constructs the URI with `file://` prefix before returning. iOS: `fileURL.absoluteString`. Android: `"file://" + destFile.absolutePath`.

---

## Build Order for v0.1

The roadmapper should derive phases from these chunks. Each is independently shippable to a `feat/` branch.

### Chunk 1: Scaffold + Types + JS API Skeleton

**Goal:** Working npm package with correct public API shape; no native code. Calling `scan()` returns `{ kind: 'error', code: 'unknown', message: 'Not implemented' }`.

**Contents:**
- `npx create-react-native-library@latest react-native-system-scanner` (turbo-module, Kotlin & ObjC template, pnpm)
- Post-scaffold: add `biome.json`, configure `tsconfig.json` strict, pin TypeScript `~5.8`
- Write `src/types.ts` — all types, zero `any`
- Write `src/NativeSystemScanner.ts` — codegen spec
- Write `src/scan.ts` — ULID generation + stub that returns error result
- Write `src/index.ts` — exports
- Add `ulidx` dependency
- Stub native: iOS `.mm` returns reject; Android `.kt` returns reject
- Biome passing, `tsc --noEmit` passing
- Jest tests for result shape and error mapping logic (pure TS, no native mock needed)

**Dependencies:** None — this is the foundation.

**Outputs depend on:** Chunks 2, 3, 4, 5 all depend on the types from this chunk being stable. Lock `ScanOptions`, `ScanResult`, and `ScanErrorCode` shapes before proceeding.

---

### Chunk 2: iOS Native — VisionKit + Swift Bridge + PDFKit

**Goal:** `scan()` works on a real iOS device; all three output formats (jpeg, pdf, both) return stable `file://` URIs.

**Contents:**
- Post-scaffold Swift wiring:
  - Add `*.swift` to `source_files` in podspec
  - Create `ios/RNSystemScannerImpl.swift` with `@objc public class`
  - Import `react_native_system_scanner-Swift.h` in `.mm`
- Implement `isSupported` → `VNDocumentCameraViewController.isSupported`
- Implement `scan` → present `VNDocumentCameraViewController` modal
- Implement all three delegate callbacks (success, cancel, error)
- Write JPEG pages to `NSCachesDirectory/scans/<scanId>/`
- Implement `PDFComposer.swift` → `[UIImage]` → `PDFDocument` with `autoreleasepool` per page
- Implement `cleanup` → `FileManager` delete
- `pod install` passes; `xcodebuild` passes; manual test on real device

**Dependencies:** Chunk 1 (types must be stable; codegen spec drives the `.mm` protocol).

**Risk flag:** VisionKit delegate runs on the main thread — ensure `resolve`/`reject` are dispatched correctly. PDFKit memory on >10-page scans — test on device, not simulator.

---

### Chunk 3: Android Native — ML Kit + ActivityEventListener + File Copy

**Goal:** `scan()` works on a real Android device and on Android emulator with Play Services.

**Contents:**
- Implement `SystemScannerModule.kt` extending codegen-generated spec, implementing `ActivityEventListener`
- `init` block: `reactContext.addActivityEventListener(this)`
- Implement `isSupported` → `GoogleApiAvailability.getInstance().isGooglePlayServicesAvailable(context)` version check
- Implement `scan` → build `GmsDocumentScannerOptions`, `GmsDocumentScanning.getClient()`, `getStartScanIntent()`, `startIntentSenderForResult()`
- Implement `onActivityResult` → `GmsDocumentScanningResult.fromActivityResultIntent(data)`
- Implement `FileCopyHelper.kt` → `contentResolver.openInputStream(contentUri)` → `FileOutputStream(destFile)` for each page and PDF
- Implement `cleanup` → delete `cacheDir/scans/<scanId>/` or all of `cacheDir/scans/`
- Implement `SystemScannerPackage.kt`
- Wire `SystemScannerPackage` into example app's `MainApplication.kt`
- `assembleDebug` passes; manual test on real device and emulator with GMS
- Manual test: no-GMS emulator → `isSupported()` returns false; `scan()` returns `{ kind: 'error', code: 'unsupported' }`

**Dependencies:** Chunk 1 (codegen spec). Does NOT depend on Chunk 2.

**Risk flag:** `ActivityEventListener` wiring — `pendingPromise` must be cleared on every code path to avoid memory leak and stale promise. The `content://` copy must complete before `onActivityResult` returns — do it synchronously on the calling thread or use `runBlocking` with a bounded coroutine.

---

### Chunk 4: Expo Config Plugin

**Goal:** Expo consumers get working iOS + Android setup with one line in `app.json`.

**Contents:**
- Create `plugin/src/index.ts`
- `withInfoPlist` modifier: inject `NSCameraUsageDescription` (configurable string)
- `withAppBuildGradle` modifier: append `implementation("com.google.android.gms:play-services-mlkit-document-scanner:16.0.0")`
- `withAndroidManifest` modifier (opt-in): add `<meta-data android:name="com.google.mlkit.vision.DEPENDENCIES" android:value="docui"/>` for install-time model
- Add `plugin` export to `package.json` `exports` field
- Test in example app (Expo-managed workflow)

**Dependencies:** Chunk 3 (Gradle dependency version must be known). Chunk 2 implicitly (permission string is iOS).

**Note:** Config plugin does not affect the library's runtime behaviour — it can be built independently of Chunks 2 and 3, but testing requires both to work.

---

### Chunk 5: Example App — All Paths Demonstrated

**Goal:** The example app demonstrates every documented behaviour.

**Contents:**
- Implement `App.tsx` with screens for: image-only, PDF-only, both formats, cancel path, error path (unsupported device simulation)
- Display scanned images inline (`<Image source={{ uri }}>`)
- Display PDF page count and offer "share" action
- Show `isSupported()` result on launch; disable scan button if false
- Test matrix: 3 output formats × {iOS, Android} × {success, cancel, permission-denied}
- Manual test: Android emulator without Play Services → `{ kind: 'error', code: 'unsupported' }`
- Document test matrix results in `TESTING.md`

**Dependencies:** Chunks 2 and 3 must both be functional.

---

### Chunk 6: CI + Docs + Ship

**Goal:** Green CI; publishable package; discoverable in react-native-directory.

**Contents:**
- `.github/workflows/ci.yml`: `tsc --noEmit`, `biome check`, Jest, `pod install + xcodebuild`, `./gradlew assembleDebug`
- `README.md`: install command, 5-line example, supported platforms/versions, demo gif, non-Expo setup guide (manual `Info.plist` + `build.gradle`)
- `CHANGELOG.md`, `LICENSE` (MIT), `CONTRIBUTING.md`, `TESTING.md`
- `npm publish` (dry-run in CI, manual for actual publish)
- Submit to react-native-directory and Expo third-party module list

**Dependencies:** Chunks 1–5 all passing.

---

### Build Order Dependency Graph

```
Chunk 1 (Scaffold + Types)
    ├──► Chunk 2 (iOS native)
    ├──► Chunk 3 (Android native)
    └──► Chunk 4 (Config plugin) — can start in parallel with 2 and 3

Chunk 2 + Chunk 3 ──► Chunk 5 (Example app)
Chunk 4            ──► Chunk 5 (Config plugin tested in example)

Chunk 5 ──► Chunk 6 (CI + Ship)
```

Chunks 2, 3, and 4 can be developed in parallel after Chunk 1 is complete and types are locked. A single engineer doing this sequentially should do 1 → 2 → 3 → 4 → 5 → 6, but the types in Chunk 1 must be locked before any native work starts.

---

## Architectural Risks

### Risk 1: ActivityResultLauncher vs. ActivityEventListener on Android

**Severity:** HIGH

**What can go wrong:** ML Kit's official Android docs show `registerForActivityResult` (the modern API). TurboModules do not have a lifecycle hook that fires before the Activity reaches `CREATED`, making it impossible to call `registerForActivityResult` inside the module. Calling it late causes a crash.

**Mitigation:** Use `ActivityEventListener` + `reactContext.addActivityEventListener(this)` + `startIntentSenderForResult` (the older but fully functional path). This is the established pattern for RN native modules and is supported by RN's `BaseActivityEventListener`. The `ActivityResultLauncher` approach (registering in `MainActivity`) is also viable but forces a manual step on non-Expo consumers.

**Study reference:** `expo-image-picker`'s `RegisterActivityContracts` block only works because it uses `expo-modules-core`'s module lifecycle, which has its own Activity hook timing. Do not assume this hook is available in a plain TurboModule.

---

### Risk 2: Swift Bridging — Xcode-Managed Swift Header

**Severity:** MEDIUM

**What can go wrong:** The `react_native_system_scanner-Swift.h` header is auto-generated by Xcode from the Swift class annotations. If the module name has hyphens, Xcode converts them to underscores in the header name. Getting the import wrong causes a compile error that is confusing to diagnose.

**Mitigation:** The podspec name determines the header name. `react-native-system-scanner.podspec` with `s.name = "react-native-system-scanner"` → header is `react_native_system_scanner-Swift.h`. Verify post-`pod install` by checking `DerivedData`. Document the header name explicitly in code comments.

**Additional:** `create-react-native-library` doesn't generate Swift files. Post-scaffold steps (add `*.swift` to `source_files` in podspec, create bridging header if needed for ObjC imports into Swift) must be documented in `CONTRIBUTING.md`.

---

### Risk 3: PDFKit Memory on Large Scans

**Severity:** MEDIUM

**What can go wrong:** `PDFDocument` holds all pages in memory until `write(to:)`. A 20-page scan with full-resolution `UIImage`s can push 200–400 MB before the PDF is written. On older devices (iPhone X class), this can trigger OOM or jetsam kills.

**Mitigation:**
1. Pre-compress every `UIImage` to JPEG at 0.85 before creating `PDFPage(image:)` — reduces per-page memory by ~5x vs. raw RGBA.
2. Wrap each page's processing in `autoreleasepool { }` to release intermediate objects promptly.
3. If post-v0.1 testing reveals OOM, replace `PDFDocument` with `CGPDFContext` (`CGPDFContextCreateWithURL`) which writes page-by-page.
4. Document a realistic 10-page soft warning and ~25-page hard limit in README.

---

### Risk 4: content:// URI Expiry on Android

**Severity:** HIGH

**What can go wrong:** ML Kit returns `content://` URIs that are scoped to the scanner Activity's lifetime. If file copy is deferred to a background coroutine that runs after the Activity is fully destroyed, the `ContentResolver.openInputStream()` call will throw a `FileNotFoundException` or return null.

**Mitigation:** Copy bytes synchronously in `onActivityResult` before returning, or use `runBlocking { withContext(Dispatchers.IO) { ... } }` inside `onActivityResult`. The copy happens once per scan; the I/O blocking on the main thread is acceptable (the modal has already dismissed, the user is waiting for the promise to resolve). For very large scans (>10 pages), a bounded coroutine scope tied to the module lifecycle is acceptable — but the URI reads must complete before the scanner process terminates.

---

### Risk 5: VisionKit Unavailability on Simulator + Mac Catalyst

**Severity:** LOW (known, easy to handle)

**What can go wrong:** `VNDocumentCameraViewController.isSupported` returns `false` on iOS Simulator, Mac Catalyst, and some iPad configurations. Calling `scan()` directly without checking crashes or returns a confusing error.

**Mitigation:** Always check `isSupported` first in the Swift implementation (not just in JS). Return `{ kind: 'error', code: 'unsupported' }` rather than presenting the VC. This means the example app and CI can run on Simulator without crashing — they'll just get an unsupported result, which is correct.

---

## Integration Points

### TurboModule ↔ Codegen

Codegen reads `src/NativeSystemScanner.ts` (via `codegenConfig` in `package.json`). It generates:
- iOS: A protocol in `RNSystemScannerModuleSpec.h` that `RNSystemScannerModule.mm` must conform to
- Android: An abstract Kotlin class `NativeSystemScannerSpec.kt` that `SystemScannerModule.kt` extends

The spec file is the single source of truth. Changing it requires regenerating stubs and updating both native implementations.

### Expo Config Plugin ↔ Host App Build System

The config plugin modifies `Info.plist`, `build.gradle`, and `AndroidManifest.xml` at prebuild time. It does not run at runtime. Non-Expo consumers must perform these modifications manually (documented in README).

### Library Cache Directory ↔ Consumer App

The library owns `<cacheDir>/scans/` entirely. Consumers receive `file://` URIs pointing inside this directory. Consumers MUST call `cleanup(scanId)` after processing to avoid storage accumulation. The library does not auto-evict (by design — see cleanup semantics above).

---

## Sources

- VNDocumentCameraViewController — https://developer.apple.com/documentation/visionkit/vndocumentcameraviewcontroller
- PDFKit — https://developer.apple.com/documentation/pdfkit
- ML Kit Document Scanner Android — https://developers.google.com/ml-kit/vision/doc-scanner/android (updated Apr 22, 2026)
- ML Kit Document Scanner result API — https://developers.google.com/android/reference/com/google/mlkit/vision/documentscanner/GmsDocumentScanningResult
- React Native TurboModule structure — https://reactnative.dev/docs/turbo-native-modules-introduction
- expo-image-picker ActivityResultLauncher pattern — https://github.com/expo/expo/blob/main/packages/expo-image-picker (RegisterActivityContracts block)
- Swift TurboModule bridging pattern — https://reactnative.cn/docs/0.82/the-new-architecture/turbo-modules-with-swift
- ActivityEventListener in RN native modules — https://oneuptime.com/blog/post/2026-01-15-react-native-android-bridge/view
- ML Kit production scanner with content:// copy — https://gaikwadchetan93.medium.com/building-a-production-ready-document-scanner-in-android-with-google-ml-kit-2e76a82b99c5
- create-react-native-library templates — https://github.com/callstack/react-native-builder-bob/tree/main/packages/create-react-native-library/templates
- react-native-new-architecture turbo modules guide — https://github.com/reactwg/react-native-new-architecture/blob/main/docs/turbo-modules.md

---

*Architecture research for: react-native-system-scanner (TurboModule wrapping VisionKit + ML Kit)*
*Researched: 2026-05-04*
