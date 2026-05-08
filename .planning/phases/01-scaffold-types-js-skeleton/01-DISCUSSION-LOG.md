# Phase 1: Scaffold + Types + JS Skeleton - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-08
**Phase:** 1-Scaffold + Types + JS Skeleton
**Areas discussed:** Result type expressiveness

---

## Gray Area Selection

| Option | Description | Selected |
|--------|-------------|----------|
| Result type expressiveness | Loose optional fields vs conditional types tied to outputFormats | ✓ |
| Stub plumbing strategy | Pure JS short-circuit vs codegen+native stub on both platforms | |
| Error code taxonomy | Reconcile REQUIREMENTS API-12 (6 codes) vs research SUMMARY (5 codes); decide cancelled_by_system semantics | |
| Phase 1 test + example scope | Type-narrowing only vs runtime tests; CRNL default vs first-pass demo | |

**User's choice:** Result type expressiveness only. The other three were left to Claude's discretion (captured in CONTEXT.md §Claude's Discretion).

---

## Result Type Expressiveness

### Q1: How tightly should ScanResult.success narrow based on requested outputFormats?

| Option | Description | Selected |
|--------|-------------|----------|
| Loose: both optional | Keep current spec. images? and pdfUri? both optional; consumer null-checks. Simplest types, easiest codegen mapping. | |
| Tight: conditional by outputFormats | scan() generic over the outputFormats tuple. ['jpeg'] → images: string[], no pdfUri. Codegen spec stays flat; JS wrapper narrows. | ✓ |
| Split union | Three explicit success kinds: success-images, success-pdf, success-both. No generics. Verbose but explicit. | |

**User's choice:** Tight conditional types.
**Notes:** User accepted the recommended option. Drives D-01 in CONTEXT.md.

### Q2: How should the consumer obtain a scanId for targeted cleanup()?

| Option | Description | Selected |
|--------|-------------|----------|
| Expose on success only | scanId on the success variant; cancelled/error don't carry it. Cleanest pairing with cleanup(scanId). | ✓ |
| Expose on every kind | scanId on success, cancelled, AND error. Useful for telemetry/log correlation. | |
| Don't expose | cleanup() takes no args; always clears the entire tree. Simplest API. | |

**User's choice:** Expose on success only.
**Notes:** Drives D-02. Consumer pattern locked: `if (r.kind === 'success') await cleanup(r.scanId)`.

### Q3: How should we handle the as-const requirement for narrowing?

| Option | Description | Selected |
|--------|-------------|----------|
| Require as const, document it | Standard TS idiom (zod, tRPC, react-hook-form). README example shows `as const`. Without it, fallback to loose union. | ✓ |
| Helper export to skip as-const | defineScanOptions() builder. More API surface; clever; less standard. | |
| Output-format string param | Drop the array. Use 'jpeg' \| 'pdf' \| 'both' single literal. Loses array extensibility. | |

**User's choice:** Require as const, document it.
**Notes:** Drives D-03. README's 5-line example MUST use `as const`. TSDoc on scan() must mention this pattern with a working example.

### Q4: How should system-initiated cancellation be represented?

| Option | Description | Selected |
|--------|-------------|----------|
| kind:'error' code:'cancelled_by_system' | Match REQUIREMENTS.md API-12. 6 codes total. Distinguishes user cancel from OS kill. | ✓ |
| Collapse into kind:'cancelled' | Match research SUMMARY. 5 codes total. Consumer can't distinguish OS-kill from user back-out. | |
| kind:'error' code:'unavailable' | Reuse existing 'unavailable' for system-cancel. 5 codes, no new symbol. 'unavailable' becomes overloaded. | |

**User's choice:** kind:'error' code:'cancelled_by_system' — REQUIREMENTS.md API-12 is canonical.
**Notes:** Drives D-04. `research/SUMMARY.md` is stale on this point and should be refreshed at next `/gsd-transition` (captured in Deferred Ideas).

---

## Wrap-up

| Option | Description | Selected |
|--------|-------------|----------|
| Ready for context | Write CONTEXT.md now; remaining areas captured as Claude's discretion. | ✓ |
| More on result types | Empty-array handling, runtime validation strictness, default narrowing edge cases. | |
| Pivot to another area | Open stub plumbing / test scope / TurboModule naming / Biome strictness. | |

**User's choice:** Ready for context.

---

## Claude's Discretion

User left these to Claude — sensible defaults captured in CONTEXT.md §Claude's Discretion (CD-01..CD-06):

- **CD-01 — Stub plumbing:** native stub on both platforms (validates codegen pipeline end-to-end now).
- **CD-02 — TurboModule naming:** `SystemScanner` Spec / `RNSystemScanner` Obj-C++ / `SystemScannerModule` Kotlin / `"SystemScanner"` codegenConfig.name.
- **CD-03 — Phase 1 test scope:** type-narrowing tests + ULID + options-normalisation + flat-result→typed promotion. No native mocking.
- **CD-04 — Phase 1 example app:** CRNL scaffold + one button that calls scan() and renders JSON. No demo screens.
- **CD-05 — Biome ruleset:** `recommended` preset + `noConsole: error`. Tighten per-rule only on real friction.
- **CD-06 — `PrivacyInfo.xcprivacy`:** deferred to Phase 2 per ROADMAP.md IOS-06. No iOS disk access at stub stage.

Each is open to redirect at plan-review.

---

## Deferred Ideas

- `PrivacyInfo.xcprivacy` belongs to Phase 2 (already in ROADMAP.md as IOS-06).
- `research/SUMMARY.md` error-code paragraph is stale — refresh at next `/gsd-transition` to make it 6 codes.
- V2-03 per-page metadata: `images: string[]` is non-breaking-extensible via a future `pageMeta?: PageMeta[]` field.
- `defineScanOptions()` helper: rejected for v0.1; re-open only if `as const` ergonomics surfaces as real friction.
