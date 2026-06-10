# SPIKE_PLAN — Modules as EAS Updates, Feasibility Spike

**Companion to:** `IMPL_GENERIC_1_EXPO_NATIVE.md` (long-term architecture / vision doc)
**Status:** Plan — not started
**Timebox:** 4 weeks, iOS only
**Decision this spike informs:** go / no-go on building the module framework on Expo machinery (EAS Update branches as registry, eval-loaded JS modules in the shared Hermes runtime).

## 1. Context and framing

This is a feasibility spike, not phase 1 of a committed roadmap. Decisions already made (from spec interview, 2026-06-10):

- **Authors:** internal AI agents + employees only. Soft (capability-facade) isolation is acceptable for now; every module passes human review.
- **Scale target:** one host app, 10–50 modules in year one.
- **EAS coupling:** full EAS, exit hatch stays on paper (open protocol + reference server noted, not exercised).
- **Sidecar native work:** deferred entirely. The spike runs on a JS-only loader. Sidecar tractability is de-risked by the code-reading assessment in Appendix A, not by building it.

## 2. Hypotheses under test

The spike attacks the two assumptions that kill the approach if false:

- **H1 — Runtime loading works at all.** A Metro bundle built with externals can be fetched, eval'd into the shared Hermes runtime, share the host's single React instance, mount through an expo-router catch-all route, and survive normal app lifecycle (backgrounding, reload, repeated mount/unmount) without being a demo-only hack.
- **H2 — EAS-as-registry holds up.** A module published with stock `eas update` to a dedicated branch can be fetched by the app itself over the open expo-updates protocol (plain `fetch` against `u.expo.dev`, multipart parsing, base64url SHA-256 verification) *without* the native expo-updates client launching it — and nothing about the platform fights this usage.

The third kill-risk — **H3, sidecar fork tractability** — is addressed by evidence, not experiment: see Appendix A. Verdict: tractable (the components are public APIs, not internals; precedent exists in expo-dev-launcher).

## 3. What is REAL in the spike

| Piece | Spike implementation |
|---|---|
| Module authoring | Standalone workspace with custom Metro config: `resolver.resolveRequest` rewrites externals (`react`, `react-native`, `@acme/module-sdk`) to a shim reading `global.__HOST_MODULES__` |
| Module bundle | Plain-JS IIFE registering into `global.__MODULE_REGISTRY__`. Built with `npx expo export --no-bytecode` — **EAS publishes Hermes bytecode by default, which cannot be eval'd**; plain-JS output is mandatory (verify exact flag plumbing in week 1: `expo export --no-bytecode` then `eas update --skip-bundler --input-dir dist`, or equivalent) |
| Publish | Stock `eas update` to branch `module/demo/dev` on a dedicated EAS project |
| Fetch | JS-only protocol client inside the host app: correct request headers (`Expo-Protocol-Version: 1`, `Expo-Runtime-Version: host-sdk@1`, `Expo-Channel-Name`), multipart/mixed response parsing, 204 no-update handling, base64url SHA-256 hash verification of every asset. Checklist + file references in Appendix B |
| Caching | Downloaded bundles + verified hashes persisted via expo-file-system; offline relaunch loads from cache |
| Loading | `eval` with `//# sourceURL=module://demo/<version>` (use `globalEvalWithSourceUrl` only if present — repo search suggests it is not generally available; do not depend on it) |
| SDK | Minimal `@acme/module-sdk`: `nav` (push/back via expo-router), `net` (fetch scoped to an allowlisted host), `storage` (namespaced key-value). Frozen facade object handed to the module factory |
| Mounting | One catch-all route `app/mod/[...slug].tsx` + `<ModuleErrorBoundary>` + try/catch around eval. No watchdog, no crash-loop counter |
| Demo module | Toy feature: list screen → detail screen, fetches a public API, persists a counter/favorite via `storage` — exercises all three SDK capabilities |
| Dev hot-load | In dev builds the ModuleLoader also accepts `http://localhost:8081/module.bundle?platform=ios` from the module workspace's Metro — same loader and SDK path as the EAS-fetched bundle. This proves the sandbox-parity story |

## 4. What is deliberately FAKED or deferred

| Piece | Spike stance |
|---|---|
| Code signing | Deferred — hash check only. Honest reason: RSA verify of the `expo-signature` header in pure JS on Hermes needs RFC 8941 parsing, PEM/X.509 handling, and crypto Hermes doesn't provide; production signing belongs in the native sidecar where `CodeSigningConfiguration` already does it (Appendix A.4) |
| Composition service | Static JSON (module list + a boolean kill-switch flag) bundled with the host / fetched from a gist. No Worker, no per-user targeting |
| Sidecar (`expo-module-host`) | Not built. Appendix A is the deliverable |
| Android | Out of scope; assumed symmetric until proven otherwise |
| Hostile-module containment | Only ErrorBoundary + eval try/catch. No crash-loop auto-disable, no render watchdog |
| Module assets (images/fonts) | Stretch goal only (week 4) — resolve one image via the asset key → local file URI mapping. Not a go/no-go criterion |
| Promotion pipeline, AI review, capability lint | All deferred — paper design only in the vision doc |

## 5. Week-by-week plan

**Week 1 — Loader mechanics on the happy path (H1 core).**
Host shell: fresh Expo app, expo-router, expo-dev-client build. Module workspace with Metro externals config; verify the `resolveRequest` shim pattern produces a small (tens of KB) plain-JS bundle. Load the bundle from a local file (no network yet), eval, mount via catch-all route. **Checkpoint: module renders, one React instance confirmed (e.g. hooks from module code work against host React), bundle size sane.**

**Week 2 — Real EAS round trip (H2).**
Dedicated EAS project; publish the module with `eas update` to `module/demo/dev` (resolving the no-bytecode plumbing). Implement the JS protocol client (headers, multipart parse, hash verify, 204 handling — Appendix B). Host fetches, caches, evals the same module from `u.expo.dev`. **Checkpoint: cold start → module loaded from CDN; airplane-mode relaunch → module loaded from cache.**

**Week 3 — SDK + dev hot-load + lifecycle abuse.**
Minimal SDK (nav/net/storage) as frozen facade; rebuild the toy module against it. Dev hot-load path from workspace Metro into the dev-client host. Lifecycle abuse pass: background/foreground, host JS reload, mount/unmount the module 50×, publish a new module version and pick it up without reinstalling the host. **Checkpoint: edit-module → see-it-in-host loop under ~30s; no leak/crash from repeated mounts (instrument memory roughly).**

**Week 4 — TestFlight proof + writeup (+ stretch).**
TestFlight build of the host (store-track binary, not dev client); module loads OTA on it. Measure: time-to-interactive for the module on cold and warm cache, eval parse cost on the oldest device available. Stretch: one image asset via localAssets-style mapping; second toy module to confirm two modules coexist. Write the go/no-go memo against the criteria below; update the vision doc's validation markers.

## 6. Go / no-go criteria

**GO if all of:**
1. Module published via `eas update` loads and renders in a TestFlight host build, fetched by the JS protocol client (H2).
2. One React instance: module uses host React/hooks with no dual-renderer hazard, and the externals shim is stable across a Metro/Expo SDK patch bump within the spike (H1).
3. Cold-cache module time-to-interactive ≤ 2s on a mid-tier device; warm-cache ≤ 500ms; eval of a ~50KB bundle doesn't visibly jank the UI thread.
4. 50× mount/unmount and host JS reload produce no crash and no unbounded memory growth.
5. Dev hot-load loop (edit in workspace → render in host) under ~30s and uses the identical loader/SDK path.

**NO-GO / rethink signals:**
- Plain-JS export path is blocked or fragile on EAS (forced bytecode) — would push toward the sidecar-first or different-bundler approaches.
- Externals shim breaks in ways that look chronic, not fixable (e.g. Metro internals resisting the resolveRequest redirect for core packages).
- Protocol fetch from `u.expo.dev` is blocked for non-launching clients (unexpected server behavior, header gymnastics, ToS issue surfaced).
- Eval parse cost or memory behavior is unacceptable on low-end hardware with no plausible mitigation short of HBC precompile (which requires the sidecar — would reorder the roadmap, not necessarily kill it).

**Explicitly NOT answered by this spike:** security against a hostile module, Android parity, signing UX, composition/targeting design, agent-authoring DX quality, behavior at 50 modules. These stay open in the vision doc.

## 7. Deliverables

- `module-host-spike/` repo(s): host shell + module workspace + JS protocol client.
- Go/no-go memo with measurements.
- Updated `IMPL_GENERIC_1_EXPO_NATIVE.md` validation markers.
- This document's appendices serve as the standing evidence base for the sidecar decision.

---

## Appendix A — Sidecar (`expo-module-host`) tractability: code-reading assessment

Assessment of `packages/expo-updates` (iOS, current main, 100% Swift) for the post-spike plan of a native module that downloads/verifies/caches N module updates without launching them. **Overall verdict: tractable, and cheaper than the vision doc assumed — most components are public API reuse, not a fork.**

| Component | Seam quality | Evidence |
|---|---|---|
| Controller architecture | **Clean.** Three controllers already coexist (`EnabledAppController`, `DisabledAppController`, `DevLauncherAppController`) selected in `AppController.initializeWithoutStarting()`. A fourth, module-host-shaped facade slots in without touching them | `packages/expo-updates/ios/EXUpdates/AppController.swift:199-376` |
| `FileDownloader` | **Clean.** `public final class`; constructor takes `(config, urlSessionConfiguration, logger, updatesDirectory, database)` — instantiable standalone with a module-specific `UpdatesConfig` (own URL/runtimeVersion/scopeKey) | `ios/EXUpdates/AppLoader/FileDownloader.swift:62,87-115` |
| `UpdatesDatabase` | **Clean.** Designed for multiple scopes in one DB (Expo Go pattern); every query takes a `config` and filters by `scope_key`. Option A: shared DB, per-module scopeKeys. Option B: second instance in its own directory (needs its own Reaper) | `ios/EXUpdates/Database/UpdatesDatabase.swift:49-115` |
| Code signing | **Clean and self-contained.** `CodeSigningConfiguration.validateSignature(signature, signedData, certChain) → SignatureValidationResult` — pure function of manifest bytes; no controller/DB entanglement. ~300 LOC across 5 files | `ios/EXUpdates/CodeSigning/CodeSigningConfiguration.swift` |
| `AppLoader` / `RemoteAppLoader` | **Mostly reusable.** Fetch-and-store orchestration is not coupled to "this becomes the main bundle"; launch assumptions live in callers (StartupProcedure/RelaunchProcedure). `launchedUpdate` is only used to enable bsdiff patching — fine to skip for modules | `ios/EXUpdates/AppLoader/RemoteAppLoader.swift` |
| Launch vs fetch split | **Clean.** `AppLauncher*`, startup/relaunch procedures, and the React bridge handler are entirely separate from the fetch stack — the sidecar never touches them | `ios/EXUpdates/AppLauncher/` |
| Selection policy | **Pluggable.** Small protocol with multiple existing implementations (filter-aware, single-update, dev-client) via `SelectionPolicyFactory` | `ios/EXUpdates/SelectionPolicy/` |
| Multipart/protocol parsing | **Clean.** `UpdatesMultipartStreamReader` (~170 LOC, public) is standalone | `ios/EXUpdates/Multipart/UpdatesMultipartStreamReader.swift` |
| Precedent | expo-dev-launcher drives update fetching with **zero internal imports** — protocol-based via `UpdatesControllerRegistry` / `UpdatesDevLauncherInterface` from `expo-updates-interface`. The sidecar can follow the same mechanism | `packages/expo-dev-launcher/ios/ReactDelegateHandler/ExpoDevLauncherReactDelegateHandler.swift` |

**What must be vendored (internal access level):** `CertificateChain` (~230 LOC), `SignatureValidationResult` (~30 LOC), `ResponseHeaderData` and assorted error/logging helpers — roughly **500 LOC copied**, against ~6,000 LOC reused in place through public API. New code estimate: 1,500–2,000 LOC (module-host controller + orchestration).

**Consequences for the vision doc:** the risk table entry "forking expo-updates internals tracks upstream poorly" downgrades from the top risk to a managed one — the exposure is ~500 vendored LOC plus public-API stability, with the dev-launcher protocol pattern as the blessed integration route. The realistic risks shift to (a) Expo not guaranteeing semver stability of these public Swift classes across SDK majors, and (b) the same assessment not yet done for Android (`FileDownloader.kt` etc. exists but seams unverified).

## Appendix B — JS-only protocol client: implementation checklist

What the spike's in-app client must implement, with repo references. Authoritative spec: https://docs.expo.dev/technical-specs/expo-updates-1/. A full TypeScript reference server lives in-repo at `packages/expo-updates/e2e/fixtures/project_files/maestro/updates-server/server.ts`.

**Request** (easy — copy header set from `ios/EXUpdates/AppLoader/FileDownloader.swift:454-476`):
`Accept: multipart/mixed,application/expo+json,application/json` · `Expo-Protocol-Version: 1` · `Expo-API-Version: 1` · `Expo-Platform: ios` · `Expo-Runtime-Version: host-sdk@1` (arbitrary strings are fine; matching is exact-string, server-side) · `Expo-Channel-Name: <branch channel>` · `Expo-Updates-Environment: BARE` · `EAS-Client-ID: <uuid>`.

**Response** (medium): `multipart/mixed; boundary=…` with parts named `manifest`, optional `directive`, `extensions`, `certificate_chain`; `204` means no update. No JS multipart parser exists in the repo — write one (~170 LOC equivalent; algorithm in `UpdatesMultipartStreamReader.swift`).

**Manifest** (easy): `{ id, createdAt, runtimeVersion, launchAsset, assets[], metadata, extra }`; module manifest rides in `extra.expoClient.extra` (from the module's app.json `extra`). Each asset: `{ url, key, hash, contentType, fileExtension }`.

**Hash verification** (easy): `base64url(SHA-256(bytes))` — RFC 4648 §5: strip `=` padding, `+`→`-`, `/`→`_`. Native reference: `ios/EXUpdates/UpdatesUtils.swift:186-198`.

**Bundle format** (critical): default EAS/Hermes publishes `.hbc` bytecode, which **cannot** be eval'd from JS. The module export must produce plain JS (`expo export --no-bytecode`). Hermes `eval()` of plain source works (parse cost, no JIT). `globalEvalWithSourceUrl` is not reliably present — use `eval` + `//# sourceURL=` comment.

**Code signing** (hard — deferred in spike, by design): `expo-signature` is an RFC 8941 structured field `sig=…, keyid=…, alg="rsa-v1_5-sha256"` over the raw manifest-part bytes; verification needs PEM/X.509 parsing and RSA-PKCS1v15-SHA256, which Hermes lacks natively. This is a standing argument for the native sidecar in production (Appendix A.4 shows the native verifier is reusable as-is).
