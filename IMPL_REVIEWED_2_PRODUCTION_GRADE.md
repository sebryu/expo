# IMPL_REVIEWED_2 — Production-Grade Robustness Proposal

Implements `REVIEWED_SPEC.md`. Respects decided constraints: soft isolation in shared Hermes (R1–R4), plain-JS Metro bundles (C3), leaf modules only (M3), one-click human promote (P8), thin custom registry (B1).

## 1. Summary and what makes this approach distinct

This proposal treats the system as **production infrastructure from day one**: the question it optimizes is not "how do we load a module?" but "what happens at 2 a.m. when module v1.3.2 crash-loops on 4 % of Androids?" Within the spec's soft-isolation decision, it pushes containment to its hardest credible form — frozen/proxied SDK facades, native JS-thread watchdog, persisted crash-loop breakers, per-module memory/error budgets, and eval-time global-tamper detection — while staying honest (per R3/K5) that these are *bug containment and detection*, not a security boundary. It adds a complete signing/key-management story (KMS, two-key model, rotation procedure), health-gated percentage rollout with automatic halt (exceeding the MVP slice deliberately, allowed by N6 "post-MVP"), per-module crash attribution via `module://` source URLs, written disaster runbooks, and a phase-3 migration to hard isolation that changes the loader transport, never the module contract (R4). Cost honesty: this is the most expensive of the three proposals — roughly +40 % effort over a minimal MVP — bought as operational safety.

## 2. Architecture overview

```
┌────────────── Sandbox (per agent, S1–S8) ─────────────┐   ┌───────── CI / Pipeline ─────────┐
│ module-template/  src/  manifest.json  feature-spec.md │   │ P1 tsc → P2 eslint → P3 jest    │
│ Tier1: jest + @app/sdk-mock                            │──▶│ P4 build+validate (global-diff) │
│ Tier2: dev-shell + modctl serve (local registry :8765) │   │ P5 argent smoke (video/png)     │
│        argent MCP drives simulator                     │   │ P6/P7 AI review agents          │
└────────────────────────────────────────────────────────┘   │ P8 human gate (promotion packet)│
                                                             │ P9 KMS sign → publish           │
┌───────────────── Registry backend ─────────────────┐      └────────────────┬────────────────┘
│ S3 (immutable .mpk, content-addressed) + CloudFront │◀──────────publish─────┘
│ Control plane (Lambda+DynamoDB): channel manifests, │◀── Sentry webhooks (health signals)
│ rollout engine (% buckets, gates, auto-halt),       │
│ kill/rollback API, audit log, signed manifests      │
└──────────────────────┬──────────────────────────────┘
                       │ GET /channels/prod/manifest (TLS, signed, TTL 300s + push nudge)
┌──────────────────────▼──────────────── Host app ───────────────────────────────┐
│ ModuleLoader: fetch → verify(sha256+ed25519, pinned keys) → cache (FileSystem)  │
│ ModuleGuardian per mount: ErrorBoundary + native JS-thread watchdog +           │
│   crash-loop breaker (MMKV) + memory/error budgets (HermesInternal stats)       │
│ SdkBroker: capability-scoped, deep-frozen, Proxy-wrapped SDK facade per module  │
│ Mounts: route mounts (expo-router slots) + extension slots (H7)                 │
│ Observability: Sentry tags module.id/version; structured key=value logs         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 3. Module format & loading mechanism

**Artifact (`<id>-<version>.mpk`, a zip):** `manifest.json`, `bundle.ios.js`, `bundle.android.js` (platform-split Metro outputs), optional `assets/*.json`. Content hash = SHA-256 of the zip; ed25519 signature over `sha256 || canonical(manifest)`. Stored at `s3://registry/artifacts/<id>/<version>/<sha256>.mpk` — immutable and content-addressed (B3).

**Bundle build (Q4 answer: custom Metro config, not Re.Pack).** Re.Pack/Rspack replaces too much of the Expo toolchain for a one-app v1; instead the module template ships a `metro.config.js` whose `resolver.resolveRequest` maps host singletons (`react`, `react-native`, `expo-*`, design system, navigation, state lib — C1) to a shim that calls `hostRequire(name)`, and whose custom serializer emits one closed-over factory:

```js
globalThis.__APP_MODULE_FACTORIES__["todo-list@1.2.0"] =
  function (hostRequire, sdk) { /* metro module system inlined */ return { mounts } };
```

A second copy of any singleton in the dependency graph is a P4 build failure (C1). Allowlisted pure-JS deps (C2) are inlined.

**Manifest schema sketch (validated by zod in pipeline and host):**

```jsonc
{
  "id": "todo-list", "version": "1.2.0", "entryPoint": "bundle",
  "requiredSdkVersion": "^1.3.0",                       // H3/C7
  "capabilities": ["navigation", "storage.scoped", "network:api.myapp.com", "events"],
  "mounts": { "routes": ["/todos"], "slots": ["home.card"] },
  "budgets": { "maxHeapDeltaMB": 32, "maxErrorsPerSession": 5, "maxSdkCallsPerMin": 600 },
  "artifact": { "sha256": "…", "sig": "…", "keyId": "rel-2026-01", "sizeBytes": 0 },  // B4
  "rollout": { "percent": 100, "cohorts": [] }          // reserved fields per B5
}
```

**Loading path (identical dev→prod, S3/Tier-2 requirement):** fetch channel manifest → resolve compatible versions (semver check H3) → download `.mpk` via `expo-file-system` → verify sha256 + ed25519 (`@noble/ed25519`; pure JS, once per artifact) against the public key pinned in the binary → unpack to cache dir → on first navigation to a mount (H10, lazy): evaluate with `globalEvalWithSourceUrl(src, "module://todo-list@1.2.0/bundle.js")`. The source URL makes every Hermes stack frame attributable to module+version — Sentry crash tagging (H11) works even for errors that escape the error boundary (async callbacks, timers).

**Containment mechanics (the hardened H5 stack), in mount order:**
1. **Crash-loop breaker:** before eval, write `{moduleId, version, phase:"eval"}` to `react-native-mmkv`; clear after mount survives 10 s. If the *app* dies in that window, next launch finds the marker, increments a counter; ≥2 → module locally disabled until a new version appears, event reported. (Same pattern as `expo-updates` embedded-rollback.)
2. **Native JS-thread watchdog:** host-native module heartbeats the JS thread (ping/pong every 2 s); a stall >5 s while a module eval/mount is in flight marks that module suspect in MMKV (JS is non-preemptible, so we detect and attribute, never abort — stated honestly).
3. **Eval tamper tripwire:** snapshot a fingerprint of key globals/prototypes before eval, diff after. Blocking in P4 (this *is* the C4 "no top-level side effects" check); sampled (1 % of sessions) in production as detection telemetry.
4. **ErrorBoundary per mount** → unmount, hide mount point (M4), report with module tag, schedule bounded retries (3, exponential), then trip the local breaker (H5).
5. **Budgets:** SdkBroker counts SDK calls/errors; guardian samples `HermesInternal.getInstrumentedStats()` heap at mount/unmount and per minute. Budget breach → `WARN` log + metric; repeated breach across sessions → local soft-disable + report. Heap attribution in a shared runtime is delta-based and approximate — documented, used as a signal, never as proof.
6. **SDK facade:** built from `capabilities` only (H2); deep-`Object.freeze`d; wrapped in a `Proxy` that throws structured `SdkCapabilityError` on undeclared namespaces, tags rejected promises with module id, and feeds the call counters. No SES lockdown of shared intrinsics in v1 (breaks too many RN libs; cost recorded for phase 3).

## 4. Sandbox & AI-agent workflow

`npx create-app-module` scaffolds the workspace (S1): `feature-spec.md` (structured — flows + acceptance criteria, per Q6 recommendation), `manifest.json`, `src/`, eslint config with `no-restricted-imports` + `eslint-plugin-import` boundary rules (S5/P2), tsconfig against the published `@app/sdk-types` package, jest + `@app/sdk-mock` (Tier 1), and the `modctl` CLI.

- **Tier 1:** `modctl test` — jest against the mocked SDK; sub-second loop; no simulator.
- **Tier 2:** `modctl serve` runs a local registry (static manifest server, port 8765, dev-key or unsigned per S4); the dev-shell app (a debug host build pointed at `dev` channel) loads the module through the full fetch→verify→eval→mount path. The agent drives and observes via argent MCP tools (A3): `describe`/`debugger-component-tree` for discovery, `gesture-*` to drive, `screenshot` to verify.

**Verification artifacts** accumulate in `artifacts/`: `junit.xml`, `lint.json`, `bundle-report.json` (size, dep tree, global-diff result), `smoke/*.png` + `smoke.mp4`, `ai-code-review.json` (P6), `ai-behavior-review.json` (P7). The pipeline tars these into a **promotion packet** — the single thing the human gate renders (P8). No write access to the host repo exists anywhere in this loop (S2): agents consume only the dev-shell binary and `@app/sdk-types`.

## 5. Backend / registry design

- **Storage:** S3 (versioned, object-lock on artifacts) + CloudFront. Artifacts immutable (B3).
- **Control plane:** AWS Lambda + DynamoDB tables `channels` (channel → per-module pinned version, kill flag, rollout state), `audit` (every promote/kill/rollback/halt: who, when, what), `health` (per module+version: crash-free rate, error-budget burn from Sentry webhooks).
- **API:** `GET /v1/channels/{ch}/manifest?host=1.4.0&install=<uuid>` (returns *signed* manifest, TTL 300 s); admin: `POST /promote | /kill | /rollback | /halt | /resume` — each is one CLI command (`modctl kill todo-list --channel prod`) and one button in the console (P10/P11).
- **Manifest signing:** the control plane signs every served manifest with an online KMS key (separate from the artifact release key). The host pins both public keys. This closes the gap TLS-only leaves: a compromised CDN cannot un-kill a module or resurrect a rolled-back version.
- **Kill-switch distribution:** manifest refresh on every app foreground (H6) *plus* an optional silent push nudge (expo-notifications data message → immediate refresh) so production kill latency is minutes, not next-foreground.
- **Rollout engine (Phase 2):** deterministic bucketing `SHA256(installId‖moduleId) % 10000` evaluated server-side; stages 1 % → 10 % → 50 % → 100 % with minimum bake times (e.g. 2 h / 12 h / 24 h) and health gates. **Auto-halt:** if module-tagged crash-free sessions < 99.5 % or error-budget burn > threshold during a stage, the engine sets the stage to 0 % (previous version serves again — rollback is just version repointing), records the halt in `audit`, and pages the owner. MVP ships the schema and per-channel targeting only (B5); the engine lands in Phase 2.

**Disaster runbooks (written, versioned in repo `ops/runbooks/`):** RB-1 module crash-loops in prod (kill → rollback → confirm via health table); RB-2 registry/control-plane outage (hosts serve from cache per H9; embedded baselines per H8; degraded = frozen module set, app fully functional); RB-3 bad SDK release in host (modules gated by `requiredSdkVersion`; rollback host via store/EAS, modules unaffected); RB-4 suspected key compromise (kill all non-embedded modules — fail-safe because verification failure means "don't load", then execute rotation procedure §7); RB-5 auto-halt fired (triage health table → either `resume` after a false positive or `rollback` + open a fix cycle in the sandbox).

**Health signals & log schema:** every module-attributed event ships as structured key=value (H11): `level=error module=todo-list version=1.2.0 phase=render boundary=home.card err=…`. Sentry tags `module.id`/`module.version` come from error-boundary attribution *and* `module://` stack-frame matching, so async/timer crashes attribute correctly. The control plane's `health` table aggregates crash-free-session rate and error-budget burn per module+version+host-build — the exact inputs the rollout gates and the owner's console read.

## 6. Verification & promotion pipeline

GitHub Actions on the module workspace; every stage pre-human fully automated (D6).

| Spec | Concrete tool |
|---|---|
| P1 typecheck | `tsc --noEmit` against `@app/sdk-types` |
| P2 lint/boundaries | eslint + `no-restricted-imports`, `eslint-plugin-import`; allowlist check (C2) |
| P3 unit tests | jest, Tier-1 harness |
| P4 build/validate | `modctl build`: Metro bundle, zod manifest validation, capability/mount sets checked against host-exported JSON, singleton-duplication check, **eval in bare Hermes CLI with global-diff tripwire** (side-effect check, C4) |
| P5 smoke test | macOS runner: boot simulator, install dev shell, `modctl serve`, drive declared flows from `feature-spec.md` via argent MCP; screenshots + video; any error-boundary trip blocks |
| P6 code review | Claude agent (claude-agent-sdk) over source + manifest + capability usage; checklist includes the K3 store-compliance item; structured JSON report |
| P7 behavioral review | Claude agent gets feature-spec + P5 recordings, MAY re-drive the simulator; pass/fail-with-findings JSON |
| P8 human gate | Promotion console (small Next.js admin) renders the promotion packet; **one click** |
| P9 promote | Pipeline role calls `kms:Sign` (release key), uploads `.mpk`, updates channel manifest; staging-first is per-module owner choice (Q5 left as MAY, default-on in console) |
| P10/P11 ops | `modctl kill/rollback` + console buttons (instant, no rebuild) |
| P12 feedback | Sentry webhook → `health` table → console + Phase-2 auto-halt |

## 7. Security & app-store compliance

**Signing & keys.** Two ed25519 keys in AWS KMS, non-exportable, sign-only: `release` (artifacts; usable only by the promote pipeline role, invoked only by the human-gate click) and `manifest` (online, control plane). A third **dev key** lives in the sandbox template and is accepted only by debug host builds (S4). Both prod public keys + key ids pinned in the host binary (K6); manifests/artifacts carry `keyId` (B4). Rotation infra is deferred (N7) but the *procedure* is written now: ship host N+1 pinning old+new keys → re-sign manifests/new artifacts with new key → after old hosts age out, retire old key. Audit log on every KMS use.

**Capability enforcement depth (honest, per R3/K5):** static (P2/P4 validation of `capabilities` vs. SDK usage) is the enforcement; the frozen/proxied facade raises the runtime bar against *bugs* (undeclared-namespace access throws, tagged); direct `globalThis`/`fetch` access by a determined author is **not** prevented in v1 — detected partially by tripwire + P6 review. Security docs say exactly this. `network` capability is parameterized with a domain allowlist enforced inside `sdk.network` (the sanctioned path).

**Threat-model coverage:**

| Threat | Defense | v1 strength |
|---|---|---|
| Buggy module crashes host (primary, D3) | Boundary + watchdog + crash-loop breaker + budgets | Strong |
| Module corrupts shared state | Namespaced storage (C6), frozen facade, tamper tripwire | Medium (detect, not prevent) |
| Compromised CDN/registry | TLS + sha256 + ed25519 artifacts **and** signed manifests, pinned keys (K4) | Strong |
| Stolen/abused signing path | KMS non-exportable, human-gated sign, audit, kill-switch, RB-4 | Medium-strong |
| Malicious module author | Out of scope (N2); partial via P6 review | Weak — documented |
| Capability overreach | Pipeline validation + Proxy throw; globals uncovered | Medium — documented |
| Store-policy breach | JS-only (K1, same footing as EAS Update/CodePush), K3 review item, kill-switch as remediation (K2) | Strong process control |

## 8. Versioning / compatibility contract

SDK surface is semver (C7): additive → minor, breaking → major (rare, batched). Host advertises `sdkVersion`; loader rejects unsatisfied `requiredSdkVersion` and hides those mounts (H3, M4). Host-singleton upgrades (React, navigation — A4) only ship behind an SDK version bump; the pipeline re-runs P5 smoke for all published modules against a new host build before its release (compatibility regression gate).

**Phase-3 hard-isolation migration (R4, Q7):** trigger = first third-party/untrusted module author, or a tamper-tripwire incident in production. The contract is already shaped for it: modules receive an injected SDK object (never import host internals), all SDK APIs are async or event-shaped, mounts are declared data. Migration = loader changes only: (1) evaluate module in a second Hermes instance (worker via JSI, e.g. a `react-native-worklets`-style runtime owned by the host); (2) SdkBroker becomes an RPC proxy implementing the identical `@app/sdk-types` interface over a message channel; (3) UI mounts render via a host-side component tree driven by serialized element descriptions (Server-Components-style) **or** the module's screens are re-hosted — the costly part, isolated to the loader/renderer. Published modules' manifests, capabilities, and entry-point semantics do not change; modules rebuild against the same types. The facade-Proxy from v1 is the seam: it already mediates every SDK call, so swapping its backend from direct calls to RPC is mechanical.

## 9. Roadmap (3 phases, rough effort — 1 senior eng + AI agents)

- **Phase 1 — Hardened MVP (7–9 weeks).** Spec §7.5 slice *plus* the full containment stack (crash-loop breaker, watchdog, tripwire, frozen Proxy facade), KMS two-key signing, signed manifests, Sentry module tagging via `module://` source URLs, `modctl` CLI, module template + Tier 1/2, pipeline P1–P5 + manual P8/P9 console-less promote, runbooks RB-1..4. (~2–3 weeks over a minimal MVP — the cost of the angle.)
- **Phase 2 — Operate at cadence (5–6 weeks).** AI review agents P6/P7, promotion console with one-click gate, percentage rollout engine + health gates + auto-halt, push-nudged kill distribution, budgets enforcement loop, basic per-module dashboard (Sentry + health table), compatibility regression gate.
- **Phase 3 — Hard isolation + extraction (8–12 weeks, triggered per Q7).** Worker-Hermes runtime + RPC SdkBroker backend, renderer strategy decision, SES-style intrinsic hardening evaluation, key-rotation automation (N7), framework extraction groundwork (G6).

## 10. Key risks and mitigations

1. **Custom Metro serializer is bespoke plumbing** (K9) and can break on Expo SDK upgrades. *Mitigation:* pin Metro via Expo SDK, contract-test the serializer output (golden bundles), keep Re.Pack as a documented fallback; the artifact format hides the bundler choice.
2. **Watchdog/budget false positives** disable healthy modules. *Mitigation:* conservative thresholds, soft-disable is local + self-heals on new version, every breaker trip is reported and visible in the console; thresholds tunable via manifest without host release.
3. **Soft isolation overtrust** — someone treats the frozen facade as a sandbox. *Mitigation:* R3/K5 language embedded verbatim in SECURITY.md and the P6 reviewer prompt; tripwire telemetry quantifies the real-world gap; Q7 trigger codified.
4. **Operational surface exceeds a single owner** (rollout engine, KMS, runbooks). *Mitigation:* everything has a one-command CLI path; auto-halt defaults fail safe (serve previous version); Phase-2 console reduces routine ops to buttons; effort honesty above — Phase 2 can be deferred per module count.
5. **AI reviewer rubber-stamping** weakens the only pre-human code gate. *Mitigation:* P6/P7 prompts demand findings-or-explicit-attestations per checklist item; behavioral agent re-drives flows rather than trusting recordings; periodic human spot-audit of promotion packets.
6. **App-review risk despite JS-only footing** (K1 interpretation shifts). *Mitigation:* kill-switch can remove any module fleet-wide in minutes (K2), modules never alter primary purpose (K3 gate item), embedded baselines keep the app reviewable as a complete product.
