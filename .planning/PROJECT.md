# react-native-system-scanner

## What This Is

An open-source React Native library that lets RN apps scan documents using the **system-provided modal scanner UIs** on both platforms — Apple's `VNDocumentCameraViewController` (VisionKit) on iOS, and Google's `GmsDocumentScanning` (Play Services ML Kit) on Android. It is **not** a camera library; it sits at a different layer than `react-native-vision-camera` / `expo-camera` by wrapping the OS's modal scanner UIs rather than embedding camera views.

## Core Value

A 5-line `scan()` call that opens the OS-native document scanner and returns a stable result — `success` with image and/or PDF URIs, `cancelled`, or `error` — without the consumer having to reason about cache eviction, permissions, or platform divergence.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope. Building toward these. v0.1.0 hypotheses until shipped. -->

- [ ] **API surface** — Public API limited to `scan()`, `isSupported()`, `cleanup()`, and types
- [ ] **iOS scanner integration** — `VNDocumentCameraViewController` (VisionKit, iOS 13+) wrapped as a TurboModule
- [ ] **Android scanner integration** — `GmsDocumentScanning` (Play Services ML Kit) wrapped as a TurboModule
- [ ] **Output formats** — Caller chooses: image-only, PDF-only, or mixed (both)
- [ ] **iOS PDF composition** — PDFKit composes scanned `UIImage`s into a PDF when caller requests PDF output
- [ ] **Android PDF passthrough** — Use `GmsDocumentScanningResult.pdf` directly when caller requests PDF
- [ ] **Stable URI semantics** — Library copies bytes to `<libCacheDir>/scans/<ulid>/...` so URIs survive scanner activity teardown / cache eviction
- [ ] **Discriminated-union result** — `{ kind: 'success' | 'cancelled' | 'error', ... }`; cancellation is **not** thrown
- [ ] **Typed, actionable errors** — e.g. `"Play Services 21+ required for ML Kit Document Scanner. Detected: <version>. See: <url>"`
- [ ] **Capability detection** — `isSupported()` checks VisionKit availability on iOS and a sufficiently recent Play Services on Android, surfacing the gap honestly when unavailable
- [ ] **Expo config plugin** — Handles iOS `NSCameraUsageDescription`, Android Play Services scanner Gradle dependency, and an opt-in flag to switch the ML Kit model from on-demand download to install-time
- [ ] **`example/` app** — Demonstrates image-only, PDF-only, mixed output, cancellation, and graceful error on unsupported devices; tested on real iOS device, real Android device, and Android emulator without Play Services
- [ ] **CI green** — GitHub Actions: TypeScript check, Biome lint, JS tests, iOS pod install + xcodebuild, Android `assembleDebug`
- [ ] **TypeScript strict, zero `any`** — Public API uses literal unions (no string-literal enums); every exported symbol carries TSDoc
- [ ] **Manual test matrix** — 3 output formats × {iOS, Android, Android-no-GMS} × {success, cancel, denied} = 27 cases, documented in `TESTING.md` before ship
- [ ] **Repo hygiene** — `CHANGELOG.md`, `LICENSE` (MIT), `CONTRIBUTING.md`, `README.md` whose first screen (install command, 5-line example, supported platforms/versions, demo gif) earns <60-second comprehension
- [ ] **Distribution** — `v0.1.0` published to npm; GitHub release tagged with demo gif; submitted to react-native-directory and Expo's third-party module list
- [ ] **New-architecture only** — Old arch is explicitly unsupported and called out; TurboModule shape, no Fabric component

### Out of Scope

<!-- Explicit boundaries. Includes reasoning to prevent re-adding. -->

- **Custom scanner UI** — System UIs only. Wrapping them is the entire point of the library.
- **OpenCV / Vision Camera fallback** for unsupported devices — Returning `{ kind: 'error', code: 'unsupported' }` cleanly is the v0.1 answer; bringing in OpenCV would balloon scope and binary size.
- **Old-architecture support** — New-arch only is a deliberate constraint; supporting both doubles the surface and contradicts the learning goal.
- **Post-scan editing** (crop, rotate, filter) — System UIs handle this themselves; we don't compete with them.
- **OCR / text extraction** — Different library, different scope.
- **Barcode / QR scanning** — Owned by `react-native-vision-camera` and `expo-camera`.
- **Continuous / batch scanning** beyond what system UIs natively offer — System constraints are the constraints.
- **View component / embedded camera surface** — Both system APIs are modal; nothing to embed. Imperative API only.
- **iPad-specific presentation tweaks** beyond OS defaults — Verify split view works; don't customize past that in v0.1.
- **JPEG quality option** — 0.85 default is fine for v0.1; revisit in v0.2 only if requested.
- **Direct collision with `react-native-document-scanner-plugin`** — Package name must signal "system UI" (current choice: `react-native-system-scanner`) and not collide with the existing OpenCV-based plugin.

## Context

- **Repo state:** Greenfield. Working directory is `react-native-document-scanner/` (kept as-is); the published package is `react-native-system-scanner`. Git just initialized.
- **Author:** Senior RN engineer; this library doubles as a vehicle for studying RN new-architecture internals (TurboModule codegen, Activity lifecycle integration on Android under new-arch, Fabric/Turbo decisions). House style to match: Reanimated, MMKV, op-sqlite.
- **Why this exists:** Existing RN scanner libraries (`react-native-document-scanner-plugin`, `react-native-rectangle-scanner`) predate Android's ML Kit Document Scanner (released Sept 2023) and use OpenCV-based custom UIs that are objectively worse than what the OS now ships for free. No library currently wraps the *system-provided* scanner UIs on both platforms.
- **Companion docs:** Build & ship prompt at `lib-ideas/prompts/02-document-scanner-prompt.md`; idea doc at `lib-ideas/09-document-scanner.md` (treated as deferred reference — prompt is self-contained).
- **Known gotchas (anticipate, do not discover):**
  - VisionKit, not Vision — `VNDocumentCameraViewController` is in `VisionKit` despite the `VN` prefix.
  - `VNDocumentCameraViewController.isSupported` returns false on Mac Catalyst, simulators, and some iPad configs — test on real hardware.
  - iOS scanner returns `UIImage`s (not PDF); we compose with PDFKit when caller asks for PDF. Multi-page PDF composition can spike memory — encode/write/nil-out per page; for >10 pages, write incrementally if PDFKit allows or document the limit.
  - Android scanner returns `content://` URIs that expire when the scanner activity dies — **must** copy bytes to lib-controlled storage before resolving the promise.
  - ML Kit Document Scanner requires recent Play Services; stale GMS or no-GMS devices (Huawei, ungoogled emulators) will fail at runtime — `isSupported()` must check, and we don't fall back.
  - `ActivityResultLauncher` must register during the Activity lifecycle. New-arch TurboModules don't always have a clean Activity hook — study `expo-image-picker`'s new-arch wiring; do not invent.
  - `maxPages` is internally capped by ML Kit (~25); pass through whatever caller requests, let ML Kit clamp, document the realistic max.
  - `SCANNER_MODE_FULL` enables auto-capture; `SCANNER_MODE_BASE` requires manual capture. Default to `full`; document UX difference.
  - `isSupported()` ≠ "scan will succeed" — camera permission denial happens later. Surface both paths.
  - Verify `VNDocumentCameraViewController` behavior in iPad split view — should default to fullscreen modal and that's fine, but verify.
- **Reference reading before/during build:** Apple VisionKit + PDFKit docs; Google ML Kit Document Scanner docs (incl. Play Services version requirements); `react-native-document-scanner-plugin` source (note where its OpenCV approach is worse); `expo-image-picker` Android source (new-arch `ActivityResultLauncher` pattern); `react-native-mmkv` source (modern new-arch RN library house style).

## Constraints

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

## Key Decisions

<!-- Decisions that constrain future work. Add throughout project lifecycle. -->

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Wrap system scanner UIs only (VisionKit + ML Kit) | OS-shipped UIs are now better than OpenCV alternatives; no existing lib wraps both | — Pending |
| New-architecture only, no old-arch shim | Doubling surface contradicts both scope and the learning goal | — Pending |
| TurboModule only, no Fabric component | Modal system UIs — nothing to embed | — Pending |
| Discriminated-union result (`success` / `cancelled` / `error`) | Existing libs throw on cancel, which is wrong — cancellation is a normal user action | — Pending |
| Library copies output bytes to `<libCacheDir>/scans/<ulid>/...` | Android `content://` URIs expire with scanner activity; consumers shouldn't have to reason about cache eviction | — Pending |
| iOS PDF via PDFKit; Android PDF via `GmsDocumentScanningResult.pdf` | iOS scanner returns images so we must compose; Android returns PDF natively when asked | — Pending |
| ML Kit model: on-demand by default, install-time opt-in via config plugin flag | Smaller install size by default; opt-in for apps that want zero first-run delay | — Pending |
| Swift + Kotlin only; no Obj-C / Java | Match modern new-arch RN library house style (Reanimated, MMKV, op-sqlite) | — Pending |
| Biome + pnpm + react-native-builder-bob | Single fast lint tool, workspace ergonomics for `example/`, standard bundler | — Pending |
| Public API surface frozen at `scan` / `isSupported` / `cleanup` for v0.1 | Surface area discipline prevents v0.1 → v0.x feature creep | — Pending |
| Package name signals "system UI" (`react-native-system-scanner`) | Avoid collision with `react-native-document-scanner-plugin`; signals what's different | — Pending |
| No fallback for devices without GMS or VisionKit support | Honest unsupported error keeps scope tight; documenting the gap is the v0.1 answer | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-04 after initialization*
