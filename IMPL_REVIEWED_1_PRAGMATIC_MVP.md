# IMPL_REVIEWED_1 — Pragmatic MVP: Fastest Credible Path to End-to-End

Implements: `REVIEWED_SPEC.md` (authoritative). Settles Q4 (loader choice) for this proposal: **tiny custom loader + esbuild-built single-file bundles**, no Re.Pack, no EAS-Update bending, no Metro multi-bundle surgery.

---

## 1. Summary and what makes this approach distinct

This proposal optimizes for one number: **days until the first AI-built module runs on a real device through the full production loading path** (fetch → hash/sig verify → evaluate → mount → kill/rollback). Target: ~3 weeks.

Distinct choices versus sibling proposals:

- **No bundler framework.** Module bundles are built by **esbuild** (host singletons marked `external`), evaluated in the shared Hermes runtime via `new Function`. The host loader is ~500 lines of TypeScript in one package. Re.Pack/module-federation and Metro multi-bundle ID-offset machinery are explicitly *not* used in v1 — they are documented upgrade paths, not MVP dependencies.
- **The registry is a git repo + object storage. There is no backend service.** Channel manifests are static JSON on Cloudflare R2 behind its CDN; the "API" is `GET`. Publish/kill/rollback are a CLI and a GitHub Action. One-click promote = clicking **Merge** on an auto-generated promotion PR.
- **Everything the spec allows deferring is deferred** (N5–N9), each with a named trigger and upgrade path (§9, §10). Crash *tagging* (in MVP per H11/N8) is one Sentry tag, not a dashboard.
- **Boring, replaceable parts everywhere:** GitHub Actions, R2/S3, Sentry, ajv, `@noble/ed25519`, esbuild, argent MCP. Every component can be swapped without changing the module contract — the contract (manifest + global register function + scoped SDK object) is the only durable interface, per G6/C8.

## 2. Architecture overview

```
┌─────────────────────────────  OWNER's MAC (MVP infra)  ─────────────────────────────┐
│                                                                                     │
│  host repo (private)            module repos (one per feature, from template)       │
│  ├─ app/ (Expo host)            ├─ FEATURE_SPEC.md   ├─ src/index.tsx               │
│  ├─ packages/module-loader      ├─ module.json       ├─ __tests__/ (Tier 1)         │
│  ├─ packages/sdk  + sdk-types   └─ .github/workflows/verify.yml                     │
│  ├─ packages/sdk-mock                    │                                          │
│  └─ dev-shell build (.app/.apk ──────────┼── Tier 2: modctl dev (localhost:8090)    │
│      on GitHub Releases)                 │      + argent MCP drives simulator       │
└──────────────────────────────────────────┼──────────────────────────────────────────┘
                                           ▼
                          GitHub Actions pipeline (P1–P7)
                tsc → eslint → jest → build+validate → sim smoke (self-
                hosted mac runner, argent) → AI code review → AI behavioral review
                                           │ artifacts + reports
                                           ▼
              registry repo (git): channels/{dev,staging,production}.json
                  promotion PR  ──(owner clicks MERGE = one-click promote, P8)──┐
                                                                                ▼
                                              publish.yml: sign (ed25519) → upload
                                              artifact → rewrite channel manifest
                                                                                │
                       Cloudflare R2 + CDN  ◄───────────────────────────────────┘
                       /artifacts/<sha256>.js   /channels/production.json
                                           │  HTTPS GET
                                           ▼
┌────────────────────────────  HOST APP (stores binary)  ─────────────────────────────┐
│ ModuleLoader: fetch manifest → semver gate (H3) → download → sha256 + ed25519       │
│ verify (pinned pubkey) → cache (expo-file-system) → new Function(bundle) →          │
│ module calls global __registerModule__ → mount at route/slot                        │
│ Each mount: <ModuleBoundary> error boundary + circuit breaker + kill flag (H5/H6)   │
│ SDK: createScopedSdk(capabilities) — only requested groups (H2)   Sentry tags (H11) │
└──────────────────────────────────────────────────────────────────────────────────-──┘
```

## 3. Module format & loading mechanism

**Bundle format (C3).** One self-contained JS file, ES2017, IIFE, built by esbuild from the template:
`esbuild src/index.tsx --bundle --format=iife --target=es2017 --external:react --external:react-native --external:expo-* --external:@app/sdk ...`
A 20-line esbuild plugin rewrites each external import to `globalThis.__hostRequire__("react")`. The host loader doesn't care which bundler produced the file — a Metro-built bundle satisfying the same two globals is equally valid (this is the Q4 escape hatch; "Metro-compatible" is preserved as a contract property, not a build-tool mandate).

**Runtime injection path (H4, H10).**
1. On app foreground (and a 6 h timer), loader GETs `channels/<channel>.json`; caches it.
2. For each entry: skip if `killed`; skip + hide mounts if `semver.satisfies(HOST_SDK_VERSION, requiredSdkVersion)` fails (H3, M4).
3. Mount points render placeholders; on **first navigation** to a module's route/slot (H10): download artifact if not cached → `expo-crypto` SHA-256 must equal manifest hash → `@noble/ed25519.verify(signature, hash, PINNED_PUBKEY)` (B4) → `new Function('globalThis', bundleText)(globalThis)` inside try/catch.
   *Hermes supports `eval`/`Function` (runtime compile, no bytecode needed). Day-1 spike confirms this on device; fallback is the Metro `__d`-offset recipe (§10).*
4. The bundle's only sanctioned side effect (C4) is `globalThis.__registerModule__(id, register)`. The loader then calls `register(scopedSdk)` which returns `{ mounts: { 'route:settings/pets': Component, 'slot:home.cards': Component } }`.
5. `createScopedSdk(manifest.capabilities)` returns an object containing only requested capability groups; a `Proxy` throws a descriptive error on access to anything else (R3 — convention, not security; documented as such per K5).

**Error boundary / kill-switch mechanics (H5, H6).**
- Every mount is wrapped in `<ModuleBoundary moduleId version>`: React error boundary + try/catch around evaluate/register. Any throw → unmount, render nothing (slots) or a host fallback screen (routes), report to Sentry with tags `module.id`, `module.version` (H11).
- **Circuit breaker:** failure count persisted in AsyncStorage keyed `module@version`; 3 strikes → module locally disabled until a *different* version appears in the manifest.
- **Kill:** manifest refresh sees `"killed": true` → unmount now if mounted, delete cached artifact, hide mounts — within one foreground cycle (H6). Verified cache survives offline (H9).

**Manifest schema sketch** (`channels/production.json`, validated by ajv in pipeline and host):

```json
{ "schemaVersion": 1, "updatedAt": "2026-06-09T12:00:00Z",
  "modules": [{
    "id": "pet-tracker", "version": "1.2.0",
    "entryPoint": "index", "killed": false,
    "requiredSdkVersion": "^1.0.0",
    "capabilities": ["navigation", "storage.scoped", "events"],
    "mounts": ["route:home/pet-tracker", "slot:home.cards"],
    "artifact": { "url": "https://cdn.../artifacts/<sha256>.js",
      "sha256": "<hex>", "signature": "<base64-ed25519>", "keyId": "prod-2026-01" },
    "rollout": null, "cohorts": null }]}
```
`rollout`/`cohorts` are reserved nulls (B5); `keyId` enables rotation later without format breakage (B4/N7).

## 4. Sandbox & AI-agent workflow

**Template repo** `module-template` (GitHub template; S1): `module.json` skeleton, `FEATURE_SPEC.md` placeholder (S7), `src/index.tsx` with the register-function pattern, `tsconfig` against `@app/sdk-types` (published to GitHub Packages from the host repo), ESLint with `no-restricted-imports` allowing only `@app/sdk` + allowlist (S5/P2), Jest + `@app/sdk-mock`, `modctl` build/dev scripts, and `verify.yml`. Agents get **no credentials for the host repo** — core protection is GitHub permissions, nothing cleverer (S2).

**Tier 1 (S3):** `@app/sdk-mock` — in-memory scoped storage, recording fake navigation, synchronous event bus, design-system stubs. `jest` + `@testing-library/react-native`. Seconds-fast inner loop.

**Tier 2 (S3, S4):** `modctl dev` builds the bundle, serves `manifest.json` + artifact on `localhost:8090` **unsigned** (dev channel skips signature, keeps hash check — S4). The **dev shell** is the real host app (dev build, downloaded once from GitHub Releases) with a dev-settings screen to point the registry URL at localhost. Same loader, same path as production. The agent boots a simulator and drives the flows itself with **argent MCP** (`boot-device`, `launch-app`, `describe`, `gesture-tap`, `screenshot`) per A3/S6.

**Idea → candidate:** owner writes `FEATURE_SPEC.md` → creates repo from template → agent implements against SDK types → Tier 1 loop → Tier 2 self-verification via argent → push → `verify.yml` runs P1–P7 → green pipeline opens the promotion PR. The candidate artifact is the CI-built bundle (content-addressed), never an agent-local build.

## 5. Backend / registry design (thin)

- **Storage:** Cloudflare R2 bucket (`modules-registry`), public read via Cloudflare CDN. S3+CloudFront is a drop-in substitute. Artifacts at `/artifacts/<sha256>.js` — immutable, content-addressed (B3). Manifests at `/channels/{dev,staging,production}.json` (B2).
- **Source of truth:** the `registry` git repo holds the channel manifests; `publish.yml` syncs them to R2 on merge. Git gives audit log, review UI, and rollback history for free.
- **API:** none in MVP. Reads are plain HTTPS GET (TLS per K4). Writes go through the GitHub Action or `modctl` (R2 credentials only on owner machine + Actions secrets). Single-tenant, one bucket (B6).
- **`modctl` CLI** (Node, ~400 LOC): `build`, `validate`, `dev`, `publish`, `kill`, `rollback`, `promote`.

## 6. Verification & promotion pipeline

| Spec | Concrete step (GitHub Actions `verify.yml` in each module repo) |
|---|---|
| P1 | `tsc --noEmit` against `@app/sdk-types` |
| P2 | `eslint` with import-boundary + allowlist rules (same config agents see locally, S5) |
| P3 | `jest` Tier-1 suite |
| P4 | `modctl validate`: esbuild build; ajv manifest schema; check `capabilities`/`mounts` against `host-contract.json` (published per host build); dependency allowlist (Q2 starts as a literal JSON array in the registry repo); evaluate bundle in Node with instrumented `__hostRequire__`/`__registerModule__` stubs — any other top-level effect (network, timers, globals) fails (C4) |
| P5 | **Self-hosted macOS runner (owner's Mac mini)**: install dev shell, `modctl dev`, argent MCP drives the declared flows from `FEATURE_SPEC.md`; screenshots + screen recording uploaded as artifacts; crash/error-boundary trip/failed step = red |
| P6 | `anthropics/claude-code-action` job: code-review agent prompt over the diff + manifest → structured `code-review.md` (includes K3 store-compliance checklist item) |
| P7 | Second agent job: replays/reviews P5 recordings against `FEATURE_SPEC.md` → `behavior-review.md`, pass/fail-with-findings |
| P8 | Green pipeline auto-opens a **promotion PR** in `registry` repo: manifest diff + embedded reports + video links. **Owner clicks Merge — that is the one click.** |
| P9 | `publish.yml` on merge: fetch CI artifact → sign with ed25519 key (Actions secret) → upload to R2 → rewrite channel manifest. Staging-first is the owner's per-module choice of PR target branch (Q5 stays open, default `staging`) |
| P10 | `modctl kill pet-tracker --channel production` (direct R2 write, seconds) **and** a `workflow_dispatch` button for auditability |
| P11 | `modctl rollback pet-tracker --to 1.1.0 --channel production` — repoints manifest at the previous immutable artifact; no rebuild |
| P12 | Sentry issues carry `module.id`/`module.version` tags; a saved Sentry search per module is the MVP "dashboard". Automated feedback loop deferred |

## 7. Security & app-store compliance

- **JS-only (K1, M2):** esbuild externals + allowlist make bundling native bindings structurally impossible; P4 rejects anything non-pure-JS. Same legal footing as CodePush/EAS Update — interpreted JS, no primary-purpose change (checked in P6 per K3).
- **Signing (B4, K6):** ed25519 (`@noble/ed25519`, pure JS both sides). Private key exists only in GitHub Actions secrets (and one offline backup); public key + `keyId` pinned as a constant in the host binary. Rotation infra deferred (N7) but `keyId` is in every manifest entry.
- **Kill-switch (K2):** P10 path above; effective within one foreground cycle.
- **Honest posture (K5/R3):** docs state plainly that v1 capability scoping is build/review-time convention, not a runtime boundary; threat model is buggy AI code + compromised CDN (handled by TLS + hash + signature), not malicious authors.
- **Known MVP gap (documented):** the manifest itself is TLS-protected but unsigned; a CDN-level attacker could flip kill flags (not inject code — artifacts stay signed). Manifest signing is a Phase-2 line item.

## 8. Versioning / compatibility contract

- Host binary declares `HOST_SDK_VERSION` (semver). Each host build publishes `host-contract.json` (sdkVersion, exposed mounts, capability groups) consumed by P4 and the template.
- Modules declare `requiredSdkVersion` (range). Loader gates with `semver.satisfies` (H3); incompatible modules are silently hidden (M4).
- SDK evolution (C7): additive → minor; breaking → major, rare and batched; **only new native capabilities force a store release** (K7), so the host pre-embeds camera/location/secure-storage per roadmap (K8).
- Contract shape (id/capabilities/mounts/SDK-injection/global-register) is app-agnostic, so framework extraction (G6/C8) and even a hard-isolation move (R4 — swap `new Function` + direct SDK object for worker + RPC proxy behind the same SDK types) don't break published modules.

## 9. Roadmap

**Week 1 — host side.** Day-1 spike: `new Function` of an esbuild bundle in Hermes on device (kills the biggest unknown). Then: SDK v1 surface enumeration (Q1 — navigation, storage.scoped, events, network, theme; smallest set that supports the first real feature), `@app/sdk`/`sdk-types`/`sdk-mock`, `ModuleLoader` + `ModuleBoundary` + circuit breaker, dev shell build on GitHub Releases.
**Week 2 — sandbox + registry.** `module-template`, `modctl` (build/validate/dev/publish/kill/rollback), R2 bucket + registry repo + publish.yml, signing. Milestone: a *hand-written* module flows dev → staging → production → kill → rollback end-to-end on simulator.
**Week 3 — pipeline + first AI module.** `verify.yml` (P1–P5) with self-hosted mac runner, AI review jobs (P6–P7), promotion PR flow (P8–P9). Milestone: **first AI-built module on a TestFlight device, promoted with one click, then killed and rolled back as a drill.**
**Week 4 — buffer/hardening.** Second and third modules to shake out SDK gaps; Sentry tagging polish; kill/rollback runbook.

**Phase 2 (triggered, not scheduled):** manifest signing; staging-mandatory policy (Q5); Hermes bytecode artifacts (N5 — trigger: module eval time >200 ms on mid-range device); cohort/percentage rollout via reserved manifest fields + a thin Cloudflare Worker in front of R2 (N6 — trigger: first scary promote); key rotation (N7); per-module dashboards (N8); cloud sandboxes — GitHub Codespaces/Depot mac runners (N9 — trigger: >2 concurrent agents or runner contention); module data lifecycle policy on kill (Q3 — default: retain, namespaced).
**Phase 3:** hard isolation (worker + RPC behind unchanged SDK types) when third-party authors appear (Q7); framework extraction (G6); structured feature-spec format (Q6) once behavioral review prompts plateau.

## 10. Key risks and mitigations

| Risk | Mitigation / fallback |
|---|---|
| Hermes `eval`/`Function` disabled or too slow on some build config | Day-1 device spike before anything else; fallback A: Metro multi-bundle with `createModuleIdFactory` offset (known recipe); fallback B: accelerate N5 bytecode. Loader API unchanged either way |
| esbuild output diverges from Metro/Babel semantics (JSX runtime, polyfills) | Template pins `jsx: automatic`, `target: es2017`; Tier 2 + P5 run the *real* bundle in the *real* runtime, so divergence is caught pre-promote, not in production |
| Self-hosted mac runner is a single point of failure for P5 | Acceptable at A6 cadence (multiple/week); upgrade path is hosted macOS runners (GitHub or Depot) with zero workflow changes |
| Unsigned manifest lets a CDN attacker flip kill flags / hide modules | Availability-only impact (code injection still blocked by artifact signatures); manifest signing is first Phase-2 item |
| SDK v1 surface too narrow → agents blocked mid-feature | Q1 enumeration driven by the first three concrete feature specs, not speculation; additive minor releases are cheap (no store release, K7) |
| Error boundary can't catch infinite loops / async hangs in shared runtime | Documented R3 limitation; circuit breaker + kill-switch bound the blast radius to "kill the module", which is the spec's accepted containment level |
| Store review flags remote JS | Identical mechanism/footing as CodePush/EAS Update (K1); kill-switch demo ready for review responses (K2); compliance item in every P6 report (K3) |
