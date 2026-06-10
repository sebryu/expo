# IMPL_GENERIC_1_EXPO_NATIVE — Modules as EAS Updates (Maximal Reuse of Expo Machinery)

> **Status (2026-06-10): vision document, gated on a feasibility spike.** The actionable next step is **`SPIKE_PLAN.md`** (4 weeks, iOS only, JS-only loader); everything past the spike in this document is conditional on its GO verdict. Decisions fixed by interview: module authors are **internal AI agents + employees** (soft isolation acceptable, every module human-reviewed); scale target is **one host app, 10–50 modules** in year one; **full EAS coupling accepted**, exit hatch stays on paper.
>
> Claim markers used throughout: **✅ verified** — confirmed by code-reading of `packages/expo-updates` (evidence in SPIKE_PLAN.md appendices); **🧪 spike** — will be tested by the spike; **⏳ unvalidated** — believed true, no evidence yet.

## 1. Summary

Build the framework almost entirely out of machinery Expo already ships: each remote module is published as an **EAS Update on its own branch** (`module/<id>/<channel>`), and the host app embeds a small sidecar package (`expo-module-host`) that reuses expo-updates' `FileDownloader`, SQLite update database, `SelectionPolicy`, and the complete `CodeSigning` stack (RSA cert-chain verification of the `expo-signature` header) to fetch and cache N module bundles *without launching them as the main bundle*. The host shell itself keeps using vanilla expo-updates for its own OTA. Modules are plain-JS Metro bundles evaluated into the shared Hermes runtime behind a capability-scoped SDK, mounted through a single expo-router catch-all route. EAS channels/branches give us versioning, staged rollout (`--rollout-percentage`), and instant rollback (`eas update:republish` / `eas update:rollback`) for free; the only genuinely new backend piece is a ~200-line "composition" endpoint.

**What makes this approach distinct:** zero custom artifact storage, zero custom CDN, zero custom signing infrastructure, and no bespoke update client. We bend the open expo-updates protocol (each module = one update, `runtimeVersion` = host SDK contract) instead of inventing a module registry. The bet is that 80% of the hard, already-debugged problems (resumable downloads, hash verification, cert-chain code signing, cache DB, error recovery, rollout/rollback semantics) are solved inside `packages/expo-updates` and we only add a thin native facade plus a JS loader.

**✅ Update from code reading:** the "sidecar" is materially cheaper than this document originally assumed. `FileDownloader`, `UpdatesDatabase`, `CodeSigningConfiguration`, `RemoteAppLoader`, `SelectionPolicy`, and the multipart reader are all **public Swift API**, designed for multi-scope use (the Expo Go pattern) and already consumed cross-package by expo-dev-launcher through `expo-updates-interface` protocols. A sidecar is a *dependent package* reusing ~6,000 LOC in place and vendoring only ~500 LOC of internal helpers (`CertificateChain`, `SignatureValidationResult`, misc.) — not a fork. See SPIKE_PLAN.md Appendix A.

## 2. Architecture Overview

```
┌────────────────────────── HOST APP (store binary) ──────────────────────────┐
│  expo-router shell        Host SDK (@acme/module-sdk)        expo-updates   │
│  app/(core)/...           nav, storage, net, native caps     (shell OTA,    │
│  app/mod/[...slug].tsx ─► ModuleOutlet + ErrorBoundary        unchanged)    │
│         ▲                        ▲                                          │
│         │ mounts                 │ capability-gated calls                   │
│  ┌──────┴───────────────────────┴───────────────┐                           │
│  │ JS ModuleLoader: eval(bundle) → registry[id] │                           │
│  └──────▲───────────────────────────────────────┘                           │
│  ┌──────┴────────────────────────────────────────────────┐                  │
│  │ expo-module-host (sidecar reusing expo-updates API):  │                  │
│  │ FileDownloader + CodeSigning + SQLite DB + Selection  │                  │
│  │ loadModuleAsync(id, channel) → localBundlePath        │                  │
│  └──────▲────────────────────────────────────────────────┘                  │
└─────────┼────────────────────────────────────────────────────────────────---┘
          │ expo-updates protocol (HTTPS, multipart manifest, expo-signature)
   ┌──────┴──────────────┐        ┌─────────────────────────────┐
   │ EAS Update CDN      │◄───────│ Composition service (thin   │◄── app asks:
   │ branch per module:  │ points │ proxy, CF Worker ~200 LOC): │    "which modules
   │ module/chat/prod    │   at   │ user/cohort → [{id,channel, │     for this user?"
   │ module/chat/staging │        │  url, minSdk}]              │
   └──────▲──────────────┘        └─────────────────────────────┘
          │ eas update --branch module/chat/staging
   ┌──────┴───────────────────────────────────────────────┐
   │ Module workspace (per-module repo, AI-agent sandbox)  │
   │ create-module template · Metro externals · CI · eval  │
   └───────────────────────────────────────────────────────┘
```

- **Host app**: Expo app (expo-router, expo-dev-client for dev builds). All native code lives here. Ships baseline modules as embedded assets for first-run/offline.
- **Module packaging**: Metro bundle + manifest, published with the stock `eas update` CLI from the module workspace.
- **Delivery**: EAS Update CDN end-to-end; composition service only returns *pointers* (module list per user/channel), never artifacts.

## 3. Module Format & Loading Mechanism

**Bundle format.** A module is built by `npx expo export` inside the module workspace with a custom Metro config:
- `resolver.resolveRequest` rewrites externals (`react`, `react-native`, `expo-*`, `@acme/module-sdk`) to a shim that reads `global.__HOST_MODULES__[name]` — Metro has no native "externals", but a custom resolver redirect to a generated shim file is a standard, working pattern. This keeps module bundles small (tens of KB) and guarantees one React instance. 🧪 spike (week 1 checkpoint).
- Output is **plain JS** (IIFE assigning `global.__MODULE_REGISTRY__[id] = factory`), *not* Hermes bytecode: Hermes can evaluate JS source at runtime but cannot `eval` HBC. **✅ Critical, verified detail:** the default EAS/Hermes pipeline publishes `.hbc` bytecode as the launch asset — the module export *must* run with `expo export --no-bytecode` (exact `eas update` plumbing for prebuilt dirs to be confirmed in spike week 1). We accept the parse cost (small bundles, cached) and leave bytecode precompilation as a later sidecar extension.
- Assets (images, fonts) ride along as standard update assets; at runtime they resolve through the sidecar's equivalent of `Updates.localAssets` (asset-key → local file URI), which vanilla expo-updates already exposes for the main update. ⏳ unvalidated for module-scoped assets (spike stretch goal).

**Manifest.** The expo-updates protocol manifest already carries arbitrary app config via `extra.expoClient.extra` (✅ verified in `expo-manifests` types). The module's `app.json` puts the module manifest there:

```jsonc
// app.json → extra.module — lands verbatim in the signed EAS update manifest
{
  "id": "chat",
  "version": "1.4.0",
  "entryAsset": "module.js",            // key of the launch asset
  "hostSdk": "^2.1.0",                  // semver range against host SDK
  "capabilities": ["storage", "net:api.acme.com", "nav", "camera"],
  "routes": [{ "path": "chat", "title": "Chat", "tab": true }],
  "killSwitchPollSec": 300
}
```

`runtimeVersion` of every module update is set to `host-sdk@2` — arbitrary strings are valid; matching is exact-string and server-side filtered (✅ verified), so expo-updates' own selection mechanism refuses modules built for an incompatible host (Section 8).

**Runtime injection.**
1. On launch (and on poll/push), the host fetches the composition: `GET /composition?user=…&channel=…` → ordered module list.
2. `ExpoModuleHost.loadModuleAsync(id, channel)` (native, reusing expo-updates public API) checks the module DB, downloads the update if stale — full hash + `expo-signature` cert-chain verification — and returns `{ manifest, bundlePath }`. (Pre-sidecar, the spike substitutes a JS-only protocol client with hash verification but no signature check.)
3. JS loader reads the bundle (expo-file-system) and evaluates it with `eval` + `//# sourceURL=module://chat/1.4.0` (`globalEvalWithSourceUrl` is **not** reliably available in release Hermes — ✅ repo search found no general provision; do not depend on it); the IIFE registers its export factory.
4. The factory is invoked with a **frozen, capability-filtered SDK object** (only the capabilities granted in the signed manifest); it returns `{ routes, components, background }`.
5. expo-router cannot register routes at runtime (routes come from the filesystem `require.context` at build time), so the host ships one catch-all `app/mod/[...slug].tsx` that maps the slug to a module route and renders it inside `<ModuleErrorBoundary>` + a render watchdog; tab/drawer entries are generated from composition manifests. Unload = unmount + delete registry entry + (best-effort) drop SDK handle. 🧪 spike (weeks 1, 3).

A faulty module is contained by: per-module ErrorBoundary, try/catch around eval, kill-switch flag in the composition response, and crash-loop detection in the loader (2 crashes → auto-disable module, report). ⏳ crash-loop/watchdog deferred past spike; spike carries ErrorBoundary + try/catch only.

## 4. Sandbox Strategy for AI-Agent Development

- **Workspace**: `create-module` scaffolds a standalone repo per module: `src/` (module code), `app/` (a dev harness Expo app), `module.config.ts`, Metro externals config, Jest setup. Agents never get access to the host repo — containment is repo-level + capability-level, which is the only containment that actually holds for same-runtime JS.
- **Host Preview app**: an **expo-dev-client build of the real host shell** (same binary as production plus dev-launcher), published internally via EAS Build. In dev mode its ModuleLoader accepts `http://localhost:8081/module.bundle?platform=ios` — the agent runs Metro in the workspace and the preview app hot-loads the module through the *same* loader/SDK path as production. 🧪 spike (week 3, with a go criterion on loop latency). This mirrors how expo-dev-client + `expo-updates-interface` already let dev-launcher drive update loading (✅ pattern verified in `ExpoDevLauncherReactDelegateHandler.swift`).
- **Two test tiers**: (a) fast loop — Jest against `@acme/module-sdk/mock` (in-memory capability implementations, navigation stub); (b) device loop — agent drives the Host Preview in a simulator (Maestro or argent MCP tooling) and asserts on screens.
- **Parity guarantee**: the SDK package, manifest schema, capability enforcement, and eval loader are byte-identical between sandbox and production; only the bundle source (localhost vs EAS CDN) and signing requirement differ. Dev channel (`module/<id>/dev`) accepts a dev signing key that production hosts do not trust.
- **Blast radius**: agent CI can publish only to `module/<id>/dev` (EAS robot token scoped per project/branch); staging/production publishes happen from the promotion pipeline, never from agent credentials.

## 5. Backend / Registry Design

**EAS Update *is* the registry.** One EAS project ("modules"), one branch per module×channel (`module/chat/production`), updates as versions. At the agreed scale (one app, 10–50 modules → ≤150 branches across three channels) this stays manageable in the EAS dashboard. It buys, with zero code: artifact storage + global CDN, the open expo-updates protocol (`https://u.expo.dev/{projectId}` with `expo-channel-name`/`expo-runtime-version` headers — ✅ header set verified against `FileDownloader`), code-signing signature delivery, staged rollout (`eas update --rollout-percentage`, `eas channel:rollout`), instant rollback (`eas update:republish` an older group), and an audit trail in the EAS dashboard. 🧪 The core "fetch a branch's update from app code without launching it" loop is exactly what the spike proves.

**What EAS cannot do**: per-user/cohort targeting and "which modules does this app composition contain". Hence one thin **composition service** (Cloudflare Worker + KV, ~200 LOC): input user/channel/host build; output the module list with branch pointers and kill-switch flags. It stores no artifacts and signs nothing — losing it degrades to cached compositions (host persists the last good composition), not to a broken app. (Spike fakes this with static JSON; the Worker is post-GO work.)

**Exit hatch**: the expo-updates protocol is an open spec (https://docs.expo.dev/technical-specs/expo-updates-1/) with a reference server — including a full TypeScript implementation inside this repo (`packages/expo-updates/e2e/.../updates-server/server.ts`, ✅ verified); if EAS pricing/ToS ever becomes a problem, the sidecar client is pointed at a self-hosted server without changing the host or module format. Per interview decision, this hatch stays on paper — no CI smoke test of self-hosting.

## 6. Verification & Promotion Pipeline *(post-GO, all ⏳)*

1. **CI (module repo, every push)**: `tsc --noEmit`; ESLint with `no-restricted-imports` (only `@acme/module-sdk` + allowlisted pure-JS deps — fails if the bundle would touch `NativeModules`/`__turboModuleProxy`); Jest; bundle build + size budget; static capability audit (declared `capabilities` ⊇ used SDK calls, derived from typed SDK entry points).
2. **Runtime smoke**: CI boots the Host Preview in a simulator, loads the bundle through the real loader, runs the module's declared Maestro flows; crash or watchdog trip = red.
3. **AI review agents** (e.g. Claude Code in GitHub Actions): review the diff against the manifest — flag capability escalations, network endpoints outside `net:` grants, eval/Function use inside module code, data exfiltration patterns. Output is a structured verdict attached to the PR. With internal-only authors this gate plus human review is the primary security control (see Section 7).
4. **Human gate**: owner approves the PR. Merge → pipeline signs and runs `eas update --branch module/<id>/staging` with the production code-signing key (key lives only in the pipeline, never in agent CI).
5. **Promotion**: `eas update:republish --group <staging-group> --branch module/<id>/production` — the *identical, already-signed* artifact is promoted; staged rollout via `--rollout-percentage 10` then ramp.
6. **Rollback**: two independent levers — `eas update:republish` the previous group (CDN-level, minutes) and the composition kill-switch (flag flip, seconds, also covers "remove module entirely").

## 7. Security & App-Store Compliance

- **Signing**: expo-updates code signing with our own CA; host embeds the root cert (config plugin sets `updates.codeSigningCertificate`); every module manifest is RSA-verified by the existing native `CodeSigningConfiguration`/`CertificateChain` code before any byte of JS is evaluated. ✅ Verified self-contained: `validateSignature(signature, signedData, certChain) → verdict` is a pure function over manifest bytes, reusable without controller/DB entanglement. **Important consequence (✅):** verifying `expo-signature` in *pure JS* is impractical on Hermes (RFC 8941 structured fields + PEM/X.509 + RSA-PKCS1v15-SHA256 with no native crypto) — production-grade signing effectively requires the native sidecar. This pins the architecture sequencing: JS-only loader for the spike and early internal channels, sidecar before any signing-dependent threat model. Per-channel keys: production hosts trust only the production signing cert.
- **Capability scoping**: modules receive a frozen SDK facade; granted capabilities come from the *signed* manifest, not from module code. Honest caveat: same-runtime JS isolation is **soft** — a hostile bundle that passes review could reach globals. Given the agreed author population (internal agents + employees, mandatory human review), this is an accepted posture, not a gap to close now. Defense is layered: review gate + signing + externals-only bundling + lint bans, with a documented escalation path to hard isolation (separate Hermes runtime / `react-native-worklets` worker per module) **if authorship ever extends beyond the org** — that line, not module count, is the trigger.
- **Transport**: HTTPS to EAS CDN; SHA-256 of every asset verified before use (✅ hash format verified: base64url-encoded SHA-256, RFC 4648 §5 — replicable in JS for the spike, done natively by the sidecar later).
- **App-store compliance**: identical legal footing to EAS Update/CodePush — Apple guideline 3.3.1(b)-style allowance for interpreted JS run by the embedded engine, provided modules don't change the app's primary purpose, unlock store-bypassing payment features, or download native code (impossible here by construction: modules are JS-only, native ceiling enforced by the host). Review notes document the OTA mechanism as "feature updates via Expo Updates", which Apple already accepts at scale.

## 8. Versioning / Compatibility Contract

- **Mechanical enforcement via `runtimeVersion`**: host SDK major version *is* the runtime version (`host-sdk@2`). Module updates are published with that `runtimeVersion`; the client requests `expo-runtime-version: host-sdk@2`, so an incompatible module is never even downloaded — reusing expo-updates' core compatibility primitive instead of inventing one. ✅ Arbitrary runtime-version strings and exact-match server-side filtering verified.
- **Fine-grained check**: `hostSdk: "^2.1.0"` semver range in the module manifest, validated by the loader against the host's `SDK_VERSION` (covers minor-version feature needs within a major).
- **SDK stability rules**: `@acme/module-sdk` is semver-disciplined; additive = minor (host bumps `SDK_VERSION`, old modules unaffected), breaking = major (new `runtimeVersion`, modules must republish; host can dual-trust `host-sdk@2` and `host-sdk@3` branches during migration windows by querying both).
- **Native ceiling**: each capability maps to a host native module; the composition service filters modules whose `capabilities` the reporting host build cannot satisfy (host sends its capability table hash), so old binaries simply don't receive too-new modules. (Single-host-app scope keeps this matrix small; revisit if a second host app ever appears.)

## 9. Roadmap

**Phase 0 — Feasibility spike (NOW, 4 weeks, iOS only).** Fully specified in **`SPIKE_PLAN.md`**: JS-only loader, real EAS publish/fetch via the open protocol, eval + externals + catch-all mounting, minimal nav/net/storage SDK, dev hot-load path, toy module. Signing, composition service, sidecar, Android, and hostile-module containment deliberately out. Exit: explicit GO/NO-GO against measurable criteria (TestFlight OTA load, single React instance, ≤2s cold TTI, lifecycle abuse survival, <30s dev loop).

**Phase 1 — Production-grade delivery + sandbox (post-GO, 4–6 weeks, 2 eng).** Build `expo-module-host` as a dependent package on expo-updates' public API (✅ assessed tractable: ~6,000 LOC reused in place, ~500 LOC vendored internals, 1,500–2,000 LOC new — controller pattern and dev-launcher-style protocol integration both have in-repo precedent). Enable code signing with own CA (native verification — see Section 7). Ship `create-module` template + `@acme/module-sdk` (+`/mock`), Host Preview dev-client build with localhost module loading, composition Worker with kill-switch. Android seam assessment happens at the start of this phase (⏳ unverified today). Exit: an AI agent builds a module in the sandbox and it reaches staging untouched by humans except review.

**Phase 2 — Pipeline + hardening (4–6 weeks, 2 eng).** Full promotion pipeline (CI gates, AI reviewers, human approval, republish-based promotion, staged rollout). Per-module observability (Sentry tags from `__MODULE_REGISTRY__` stack-frame mapping via `sourceURL`, per-module usage metrics). Crash-loop auto-disable, offline embedding of baseline modules as host assets, dual-runtimeVersion migration support. Exit: 5+ modules in production, one exercised rollback drill.

## 10. Key Risks & Mitigations

| Risk | Impact | Status / Mitigation |
|---|---|---|
| Plain-JS export path blocked or fragile on EAS (default pipeline is Hermes bytecode, which cannot be eval'd) | Kills the JS-loader approach | **Promoted to a top spike question** (✅ default-HBC confirmed). Week-1/2 task: `expo export --no-bytecode` + prebuilt-dir publish; NO-GO signal if chronically fragile |
| Sidecar tracks expo-updates upstream poorly | Maintenance drag each SDK upgrade | **Downgraded** (✅): components are public API + multi-scope by design; only ~500 LOC vendored; dev-launcher precedent for protocol-based integration. Residual exposure: Expo doesn't semver-guarantee these Swift classes across SDK majors → pin per host SDK release |
| Android sidecar seams differ from iOS | Phase-1 estimate risk | ⏳ Kotlin code exists (`FileDownloader.kt` etc.) but seam quality unassessed; do the same code-reading pass at Phase-1 start |
| Hermes must parse plain JS at runtime (no HBC eval) | Slower module mount on low-end devices | Small bundles via externals; eval once per process, cache exports; lazy-mount on first navigation; 🧪 spike measures parse cost; later: sidecar adds ahead-of-time HBC compile on download |
| Soft isolation in shared runtime — malicious/buggy module reaches globals | Security/stability ceiling | **Accepted for current trust model** (internal authors + human review). Layered: signing + review + import lint + capability facade + crash-loop auto-disable. Hard-isolation escalation (per-module Hermes runtime) is triggered by *external authorship*, not scale |
| Production code signing infeasible in pure JS | Early channels run hash-check only | ✅ Confirmed; sequencing answer: sidecar lands before any signing-dependent rollout; until then distribution is internal/dev channels only |
| EAS coupling: unconventional use (branches as registry), ToS/pricing/targeting limits | Vendor risk | Accepted per interview (full EAS, paper exit hatch). Protocol is open; in-repo reference server verified; composition service abstracts "where modules come from" — swap endpoint, not architecture |
| Metro externals via custom `resolveRequest` is unofficial | Build breakage on Metro upgrades | Metro version pinned by Expo SDK choice; 🧪 spike includes one SDK patch bump; golden-file tests on bundle output in module template CI |
| Apple review interpretation drift on OTA-heavy apps | Distribution risk | Modules never alter primary purpose; same exposure as every EAS Update/CodePush app; baseline modules embedded so the store binary is fully functional standalone |
| expo-router can't add routes at runtime | UX constraint on deep links | Catch-all route + manifest-declared paths gives stable URLs (`/mod/chat/...`); host regenerates link entries from composition; acceptable for v1, custom navigator only if needed |
