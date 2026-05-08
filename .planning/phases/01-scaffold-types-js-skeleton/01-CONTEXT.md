# Phase 1: Scaffold + Types + JS Skeleton - Context

**Gathered:** 2026-05-08
**Status:** Ready for planning

<domain>
## Phase Boundary

Stand up the npm package shape with `create-react-native-library` (turbo-module template), lock the TypeScript public API + codegen spec so Phases 2–4 implement against a frozen contract, wire a stub `scan()` that returns a typed "not implemented" error, and establish the lint/format/test toolchain (Biome + tsc + Jest + builder-bob ESM-only).

This phase delivers shape, not behaviour. Native logic, Expo plugin, and the example demo screens are out of scope here — they belong to Phases 2, 3, 4, and 5 respectively.

</domain>

<decisions>
## Implementation Decisions

### Result Type Expressiveness (the API contract every other phase implements)

- **D-01: Conditional types tied to `outputFormats`.** `scan()` is generic over the requested formats: `function scan<F extends readonly OutputFormat[] = readonly ['jpeg']>(opts?: ScanOptions<F>): Promise<ScanResult<F>>`. Caller passing `['jpeg'] as const` gets `{ kind: 'success'; images: string[]; pageCount: number; scanId: string }` with no `pdfUri` field; `['pdf'] as const` gets `pdfUri: string` and no `images`; `['jpeg', 'pdf'] as const` gets both. **Codegen spec stays flat** (`NativeSystemScanner.ts` returns the loose union — codegen requires interfaces with stable shapes); the `scan.ts` JS wrapper is what narrows. Native returns a single flat object with all fields optional except `kind`/`pageCount`/`code`/`message`; the wrapper casts to `ScanResult<F>` after promoting from the flat shape.

- **D-02: `scanId: string` is exposed on the success variant only.** Cancelled and error variants do NOT carry `scanId` (native may have written nothing or partial bytes that the next call will clear). Consumers who want targeted cleanup pair it with the success result: `if (r.kind === 'success') await cleanup(r.scanId)`. This makes `cleanup(scanId?)` actually usable per-scan; without exposing the ID the consumer would be forced to call `cleanup()` (no-arg, nukes the whole tree).

- **D-03: Require `as const` for narrow inference; document it.** Standard TS idiom (zod, tRPC, react-hook-form). README's 5-line example uses `outputFormats: ['jpeg', 'pdf'] as const`. Consumers who omit `as const` still get a working result — TS widens `F` to `OutputFormat[]` and the result type falls back to the loose union (`images?: string[]; pdfUri?: string`). Behaviour is identical at runtime; only the type surface differs. **TSDoc on `scan()` MUST mention the `as const` pattern with a working example.**

- **D-04: `ScanErrorCode` has 6 members.** Final union: `'unsupported' | 'permission_denied' | 'unavailable' | 'model_unavailable' | 'cancelled_by_system' | 'unknown'`. `kind: 'cancelled'` is reserved for user-initiated cancel (back button / X tap). System-initiated termination (OS jetsam, host Activity destroyed mid-scan, app backgrounded with no return) maps to `kind: 'error'`, `code: 'cancelled_by_system'` so consumers can distinguish "user backed out" from "we got killed" for telemetry / retry logic. **REQUIREMENTS.md API-12 is canonical.** `research/SUMMARY.md` lists only 5 codes (omits `cancelled_by_system`) — that doc is stale on this point and should be treated as superseded by this CONTEXT.md.

### Claude's Discretion (sensible defaults — planner should propose; user can redirect at plan-review)

- **CD-01 — Stub plumbing strategy.** Default to **codegen + native stub**: wire the full TurboModule on both platforms with native methods that return the "not implemented" error themselves (Swift returns `{ kind: 'error', code: 'unknown', message: 'Not implemented' }`; Kotlin same). This validates the codegen pipeline end-to-end now (proves bridge, registration, codegen-config alignment), so Phase 2/3 just replaces method bodies. Cost: ~30 min more native scaffolding in Phase 1; benefit: catches bridge wiring bugs before Phase 2/3 are mid-flight. JS-side short-circuit is rejected because it hides registration mismatches until later phases.

- **CD-02 — TurboModule canonical name.** Default to `SystemScanner` for the Spec interface name (`NativeSystemScanner` for the codegen file per RN convention `Native<NAME>.ts`), `RNSystemScanner` prefix for Obj-C++ class symbols (avoids collision with Foundation/UIKit), Kotlin class `SystemScannerModule`. Codegen `name` field in `package.json` is `"SystemScanner"`. Document the mapping inline in `NativeSystemScanner.ts` as a comment block so the three registration points (Spec, .mm, package.json codegenConfig) stay aligned — mismatches cause silent runtime crashes in later phases.

- **CD-03 — Phase 1 test scope.** Default to **type-narrowing tests + ULID + options-normalisation tests**. Use `tsd` or `expectTypeOf` for compile-time assertions: `expectTypeOf(r).toMatchTypeOf<{ images: string[] }>()` when `outputFormats: ['jpeg'] as const`. Runtime tests cover (a) ULID generation produces a valid ULID per call, (b) options defaults (`outputFormats` defaults to `['jpeg']`, `scannerMode` to `'full'`, `galleryImportAllowed` to `true`), (c) flat-native-result → `ScanResult<F>` promotion. No native mocking — all tests are pure-TS. CI runs Jest + tsc + biome.

- **CD-04 — Phase 1 example app shape.** Default to **CRNL scaffold + one button**. Don't pre-build the Phase 5 demo screens. The example app's `App.tsx` shows a single "Scan" button that calls `scan()` and renders the JSON result on screen (so success criterion #2 — Metro starts, scan returns the typed stub — is visually verifiable). Cancellation, error UI, multi-format demo screens all wait for Phase 5.

- **CD-05 — Biome ruleset.** Default to **`recommended` preset + `noConsole: error`** (enforces QUAL-03). Don't go custom-strict in Phase 1 — recommended already covers organize-imports and `noExplicitAny` (warning). Tighten per-rule only if a real issue appears during Phase 2/3. Single `biome.json` at repo root, applies to library + example.

- **CD-06 — `PrivacyInfo.xcprivacy` deferred to Phase 2.** Per ROADMAP.md it's IOS-06 (Phase 2). Phase 1 does not need it because there's no actual iOS implementation yet — codegen-generated stub doesn't access disk or timestamps. Phase 2 must create it in the same commit as the Swift `RNSystemScannerImpl`.

### Folded Todos
*(none — `gsd-tools todo match-phase 1` returned 0 matches.)*

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project-level decisions (read all)
- `.planning/PROJECT.md` — Constraints, Key Decisions table, Out of Scope list. Locks tech stack (Swift+Kotlin only, no Obj-C/Java; TurboModule only, no Fabric; new-arch only).
- `.planning/REQUIREMENTS.md` §API + §Quality Gate — API-01..API-13 and QUAL-01..QUAL-05 are this phase's requirement scope. **API-12 is the canonical `ScanErrorCode` definition (6 codes).**
- `.planning/ROADMAP.md` §Phase 1 — Goal, Success Criteria, Requirements list.
- `.planning/STATE.md` — Pre-Phase 1 decisions: RN peer floor `>=0.82`, Android uses `ActivityEventListener` not `ActivityResultLauncher`, Phases 2/3/4 are independent post-Phase 1.

### Research (read for stack/architecture context, but treat SUMMARY.md as superseded on the error-code count)
- `.planning/research/SUMMARY.md` — Recommended stack and rationale. **Stale on `ScanErrorCode`** — lists 5 codes; D-04 above adds `cancelled_by_system` per REQUIREMENTS.md API-12.
- `.planning/research/STACK.md` — Version pins (`create-react-native-library@0.62.0`, `react-native-builder-bob@0.41.0`, Biome `2.4.x`, TS `~5.8`, `ulidx@2.4.1`).
- `.planning/research/ARCHITECTURE.md` — File layout (`src/{index,scan,NativeSystemScanner,types}.ts`); component responsibilities table is the authoritative module breakdown.
- `.planning/research/PITFALLS.md` — Critical pitfalls; for Phase 1, the relevant ones are "wrong CRNL template" (verify `codegenConfig.ios.type === 'turbo'` immediately post-scaffold) and "module name mismatch across the three registration points."
- `.planning/research/FEATURES.md` — Defer/should-have/must-have triage of API surface.

### CLAUDE.md (always loaded — read for tech stack pins and what NOT to use)
- `CLAUDE.md` §Validated Locked Decisions — TS `~5.8`, ESM-only builder-bob, Biome v2, Android ML Kit 16.0.0, iOS VisionKit, PDFKit gotchas, expo peer dep guidance.
- `CLAUDE.md` §What NOT to Use — turbo-module-mixed template, Obj-C/Java logic, Fabric component, `expo-modules-core` peer, `ulid` (use `ulidx`), ESLint+Prettier.

### External (companion prompt; treat as deferred reference, NOT canonical for v0.1)
- `../../../lib-ideas/prompts/02-document-scanner-prompt.md` — Original build-and-ship prompt. PROJECT.md notes it as a deferred reference; the prompt is self-contained but planning should follow the .planning/ artifacts above first; consult only if the planning artifacts are silent on a question.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
*(None — repo is greenfield. Only `.planning/` and `CLAUDE.md` exist outside git metadata.)*

### Established Patterns
- **House style references for new-arch RN libraries (read before scaffolding to set the bar):** Reanimated, MMKV, op-sqlite. CLAUDE.md flags these as the style match. Check Swift/Kotlin file organisation, codegen spec layout, and consumer-facing TSDoc richness when in doubt.
- **Swift TurboModule wiring pattern (manual post-scaffold):** Thin `.mm` bridge delegates to a `@objc public class` in `.swift`. Confirmed in `research/ARCHITECTURE.md` and `CLAUDE.md` §Swift Gotcha. Reference: expo-team usage; `create-react-native-library` does NOT scaffold Swift natively — selecting Kotlin+ObjC at the prompt is correct, then add Swift manually.
- **Android `ActivityEventListener` (NOT `ActivityResultLauncher`)** — locked pre-Phase 1. Phase 1 doesn't implement it but the codegen spec shape must accommodate the async result-via-listener flow (Phase 3 implementer should not need to widen the spec).

### Integration Points
- **`example/` workspace member** — pnpm-hoisted (`shamefully-hoist=true` in `.npmrc`); consumes the local `react-native-system-scanner` package via the workspace protocol. Phase 1 must verify Metro resolves both `react` and `react-native` from the workspace without "module not found" warnings (Success Criterion #5).
- **Codegen as the JS↔native contract boundary** — `src/NativeSystemScanner.ts` (the spec) generates Obj-C++ headers (iOS) and Kotlin abstract class (Android). Phase 2 and 3 implement against generated stubs; the spec is the contract Phase 1 freezes.
- **`package.json` codegen config** — `codegenConfig.name` ("SystemScanner") and `codegenConfig.type` ("modules") must match the Spec interface name. Verify `codegenConfig.ios.type === 'turbo'` immediately post-scaffold (PITFALLS.md flags this — `turbo-module-mixed` template silently produces old-arch shim).

</code_context>

<specifics>
## Specific Ideas

- **`ScanResult<F>` discriminated by const-tuple `F`** — pattern modelled on zod / tRPC / react-hook-form. README's 5-line minimal example MUST show `as const` so the narrow type comes through; consumers who don't `as const` get the loose union as a graceful fallback. Document this pattern explicitly in TSDoc on `scan()`.
- **`scanId` on success only** — drives the `cleanup(scanId)` ergonomics. Consumer pattern: `if (r.kind === 'success') { await use(r.images); await cleanup(r.scanId); }`. Show this in the README example block in Phase 6.
- **Native stub returns the "Not implemented" payload itself** — JS wrapper does NOT short-circuit. The native methods on iOS (Swift) and Android (Kotlin) each return the typed error directly. This validates the codegen + bridge pipeline end-to-end during Phase 1, so Phase 2/3 implementers replace method bodies rather than discovering registration mismatches.
- **Module-name canonicalisation** — `SystemScanner` (Spec interface), `RNSystemScanner` (Obj-C++ class prefix), `SystemScannerModule` (Kotlin class), `"SystemScanner"` (codegenConfig.name). Inline comment block in `NativeSystemScanner.ts` documents the mapping so the 3 registration points stay in sync.

</specifics>

<deferred>
## Deferred Ideas

Discussion stayed inside scope; no new capabilities surfaced. Items below are reminders of pre-existing deferred work that touches Phase 1 boundary:

- **`PrivacyInfo.xcprivacy`** — IOS-06 in Phase 2, not Phase 1 (no iOS disk access yet at stub stage). Phase 2 must create it in the same commit as the Swift implementation.
- **`stale-doc cleanup`** — `.planning/research/SUMMARY.md` lists 5 error codes; this CONTEXT.md adds `cancelled_by_system` to make 6 (per REQUIREMENTS.md API-12). After Phase 1 ships, refresh the SUMMARY.md error-code paragraph during the next `/gsd-transition` so future readers don't hit the discrepancy.
- **`scannerMode: 'base'` UX docs** — already deferred to README in Phase 6 (CLAUDE.md notes "Default to FULL; document UX difference"). No Phase 1 action.
- **V2-03 per-page metadata** — REQUIREMENTS.md §v2. The `images: string[]` shape locked here is non-breaking-extensible: v0.2 can add a separate `pageMeta?: PageMeta[]` field without breaking callers.
- **Helper export `defineScanOptions()`** — considered as an alternative to `as const` but rejected for v0.1 (extra exported symbol, less standard). Re-open only if user feedback flags `as const` ergonomics as a real friction point.

</deferred>

---

*Phase: 1-Scaffold + Types + JS Skeleton*
*Context gathered: 2026-05-08*
