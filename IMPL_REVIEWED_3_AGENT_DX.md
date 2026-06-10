# IMPL_REVIEWED_3 — Agent-DX-First Implementation Proposal

Implements `REVIEWED_SPEC.md`. Where this proposal makes a choice the spec left open (Q1–Q7), the choice is flagged inline.

---

## 1. Summary and What Makes This Approach Distinct

The system's throughput ceiling is not the loader, the registry, or the pipeline — it is **how many edit→observe→fix cycles per hour an AI agent can complete, and how much signal each cycle carries**. This proposal therefore spends its innovation budget on the agent toolchain and keeps runtime and delivery deliberately boring: a single-file Metro bundle, `eval` into the shared Hermes runtime behind an error boundary, JSON manifests on object storage.

Distinct from sibling proposals:

- **One CLI, `modkit`, is the entire agent interface.** Every command emits `--json` structured output with stable error codes and machine-applicable fix-its. An agent never parses prose.
- **The scaffold is the documentation.** `create-module` generates a workspace whose types, TSDoc, `AGENTS.md`, and `CONTRACT.md` teach the contract from context alone — no external docs lookup required.
- **Hot module swap into a prebuilt dev shell** — the host binary is built once per SDK version and downloaded; module work never touches Xcode/Gradle. Inner-loop latency target: <1 s Tier 1, <3 s save-to-screen Tier 2.
- **Reports are a data format, not documents.** Verification, AI review, and production feedback all share one JSON schema, so the output of every stage is the input of the next agent — and renders into the one-click human promote UI.

---

## 2. Architecture Overview

```
┌──────────────────────────── AGENT WORKSPACE (per feature, S1/S2) ────────────────────────────┐
│  create-module scaffold                                                                      │
│  ├ module.json  src/index.tsx  spec/feature-spec.yaml  AGENTS.md  CONTRACT.md                │
│  ├ @app/module-sdk (types) ── @app/sdk-mock (Tier 1 harness)                                 │
│  └ modkit CLI: test --watch │ dev │ check │ verify │ publish │ feedback   (all --json)       │
│        │Tier1: jest+mock          │Tier2                                                     │
└────────┼──────────────────────────┼──────────────────────────────────────────────────────────┘
         │                          ▼
         │            ┌─ modkit dev ───────────────────────┐      ┌─ argent MCP ────────────┐
         │            │ Metro (module bundle, watch)       │      │ describe / tap / swipe  │
         │            │ local registry (Hono, :8082)       │◄────►│ screenshot / logs       │
         │            │ ws "module-updated" push           │      │ flow record & replay    │
         │            └───────────────┬────────────────────┘      └───────────▲─────────────┘
         │                            ▼ same fetch→verify→eval→mount path as prod (S3 tier 2)
         │            ┌─ DEV SHELL (prebuilt host .app on simulator) ─────────┼─────────────┐
         │            │ loader │ error boundaries │ scoped SDK │ __DEVSHELL__ debug API     │
         │            └────────────────────────────────────────────────────────────────────┘
         ▼
┌─ PIPELINE (GitHub Actions) ──────────────┐   ┌─ REGISTRY (S3/R2 + CDN + Hono admin API) ──┐
│ Stage1: tsc/eslint/jest/build/manifest   │   │ artifacts/<id>/<ver>/<sha256>.bundle.js     │
│ Stage1.5: simulator smoke via argent     │──►│ channels/{dev,staging,production}.json      │
│ Stage2: AI code-review + behavior agents │   │ feedback/<moduleId>.json  (prod telemetry)  │
│ → verification-report.json (shared schema)│  │ POST /promote /kill /rollback (signed)      │
└───────────────┬──────────────────────────┘   └───────▲──────────────────┬──────────────────┘
                ▼                                      │                  ▼
        ┌─ PROMOTE UI ─────────────┐                   │      ┌─ PRODUCTION HOST APP ────────┐
        │ reports + video render   │── one click ──────┘      │ loader: fetch→hash/sig→eval  │
        │ promote / kill / rollback│                          │ →mount; kill-switch; Sentry  │
        └──────────────────────────┘                          │ tagged module.id/version ────┼─► feedback/
        Owner (human)                                         └──────────────────────────────┘    (closes loop)
```

## 3. Module Format & Loading Mechanism

**Format (C1–C3, K9):** one IIFE-style JS bundle per module, built by **Metro** with a custom `resolver.resolveRequest` that redirects host singletons (`react`, `react-native`, `expo-*`, `@app/design-system`, `@app/navigation`, `@app/state`) to a generated shim: `module.exports = globalThis.__HOST_EXTERNALS__["react"]`. One bundler everywhere (dev loop, pipeline, smoke test) — no dual-bundler skew. Allowlisted pure-JS deps (Q2 starter set: `zod`, `date-fns`, `lodash-es` subpaths, `nanoid`) are bundled in; everything else is a Stage-1 build error.

**Manifest (`module.json`, validated by `zod` schema exported from `@app/module-sdk/manifest`):**

```jsonc
{
  "id": "habit-tracker",
  "version": "1.2.0",
  "entryPoint": "dist/habit-tracker.bundle.js",
  "requiredSdkVersion": "^1.3.0",
  "capabilities": ["navigation", "storage.scoped", "events", "network"],
  "mounts": { "routes": ["habits"], "slots": ["home.card"] },
  "flows": [                                  // machine-readable smoke flows (P5, Q6)
    { "name": "create-habit", "argentFlow": "flows/create-habit.flow.json",
      "acceptance": ["habit appears in list", "card shows count on home slot"] }
  ],
  "publish": { "sha256": "…", "signature": "…", "keyId": "prod-2026-01" },   // added at publish (B4)
  "targeting": {}                              // reserved (B5)
}
```

**Loading (H4):** loader fetches `channels/<channel>.json` → for each module: semver-check `requiredSdkVersion` (`compare-versions`, pure JS) → download to `expo-file-system` cache (content-addressed; offline = cache hit, H9) → SHA-256 + **Ed25519 verify via `@noble/ed25519`** against the pinned key (B4; dev channel accepts `keyId: "dev"`, S4) → evaluate lazily on first mount with `globalEvalWithSourceUrl(code, moduleId)` inside try/catch + 2 s watchdog → call the bundle's default export with the capability-scoped SDK object → mount returned components at declared route/slot. Error boundary per mount; 3 failures of same version trips the local circuit breaker (H5). Kill flag honored on every manifest refresh, checked on app foreground (H6). Re.Pack/module federation rejected: more machinery than this needs (Q4 answered: custom Metro multi-bundle).

## 4. Sandbox & AI-Agent Workflow (the heart)

### 4.1 Scaffold: `npx @app/create-module habit-tracker`

Generates a standalone repo (S1/S2 — agent has zero host access): `module.json` pre-filled, `src/index.tsx` with a typed `defineModule()` skeleton, `spec/feature-spec.yaml` template (structured flows + acceptance criteria — Q6: **structured**, so the behavioral reviewer can execute it), `flows/` for recorded argent flows, ESLint config with boundary rules, Jest + `@app/sdk-mock`, and two context files written *for LLM consumption*: **`AGENTS.md`** (the exact loop: which command to run when, how to read `--json` errors, when to take screenshots) and **`CONTRACT.md`** (generated by `@microsoft/api-extractor` from the SDK package — every capability, every type, with examples). An agent dropped into this directory needs nothing else in context.

### 4.2 Typed SDK: `@app/module-sdk`

```ts
export default defineModule((sdk: ModuleSdk<"navigation" | "storage.scoped" | "events">) => ({
  routes: { habits: HabitsScreen },
  slots: { "home.card": HabitCard },
}));
```

`ModuleSdk<Caps>` is generic over the manifest's capability list: requesting `sdk.camera` without declaring `camera` is a **compile error**, so capability violations surface in the editor, not the pipeline (S5 at the type level). Every SDK method carries TSDoc with an `@example` — the agent learns the API from hover/typecheck output alone. Q1 v1 surface: `navigation`, `storage.scoped` (MMKV-backed, namespaced by module id per C6), `network` (host fetch with module-tagged telemetry), `events` (pub/sub), `state.shared` (read-only host selectors + module-namespaced writes), `ui` (design-system re-exports), `media.camera`, `location`.

### 4.3 Machine-readable feedback at every layer

- **`modkit check --json`** → `{ code: "MOD_E012", message: "Imported 'axios' which is not on the dependency allowlist", file, line, fixit: { replaceImport: "sdk.network.fetch" }, docsUrl }`. Stable error-code taxonomy (`MOD_Exxx`) shared by ESLint rules (custom plugin `eslint-plugin-module-boundaries`, built on `no-restricted-imports` + `import/no-extraneous-dependencies`, all rules with `--fix`), the bundler, and manifest validation.
- **Dev-shell loader errors** are pushed back over the local registry websocket as the same JSON shape (e.g. `MOD_E031 top-level side effect during evaluation: called fetch() at habit-api.ts:3`), so a Tier-2 failure lands in the agent's terminal as structured data, not as a red screen it must screenshot and squint at.

### 4.4 Inner loop

- **Tier 1 (S3):** `modkit test --watch` — Jest against `@app/sdk-mock`, an in-memory SDK with scenario fixtures (`sdkMock.storage.seed(...)`, `sdkMock.events.expectEmitted(...)`). Sub-second; where agents spend 90% of cycles.
- **Tier 2:** `modkit shell install` downloads the prebuilt dev-shell `.app`/`.apk` for the current SDK version (built by host CI, never by the agent). `modkit dev` starts Metro (module only) + a local Hono registry on `:8082` serving a `dev` channel manifest pointing at Metro. On save: incremental rebundle (~200 ms) → ws push → dev shell unmounts, re-evaluates, remounts the module **without rebuilding or restarting the host**. Same fetch→verify→eval→mount path as production (S3 requirement), minus production signing (S4).

### 4.5 Simulator automation (S6, A3)

The dev shell ships agent hooks: deep links (`devshell://mount/habit-tracker`, `devshell://reset-state`), a `__DEVSHELL__` debug API readable via argent's `debugger-evaluate`, and module-tagged console logs. The agent drives Tier 2 with **argent MCP** (`describe`/`debugger-component-tree` → `gesture-tap` → `screenshot`), records each happy path once with `argent-create-flow`, and commits it to `flows/` — the same flow file the pipeline replays in P5. Writing a flow is writing a test.

### 4.6 Structured production feedback (closes the loop)

`modkit feedback habit-tracker --json` fetches `feedback/<moduleId>.json` from the registry (§5): crash groups (symbolicated via the sourcemap uploaded at publish), error-boundary trips, kill/rollback history, usage counts per flow. The JSON includes `suspectedFiles` and the offending release version so a maintenance agent can be pointed at the module repo with the feedback file as its only briefing and produce a candidate fix.

## 5. Backend / Registry Design

Thin and boring (B1): **Cloudflare R2 (or S3) + CDN** for immutable, content-addressed artifacts and per-channel JSON manifests; one small **Hono** admin service (single-tenant, owner-token auth) exposing `POST /promote`, `POST /kill`, `POST /rollback`, `GET /channels/:c`, `GET /modules/:id/feedback`. Rollback = rewrite one entry in the channel manifest to a prior artifact URL (B3, P11) — O(seconds), no rebuild. Signing: Ed25519 private key in the pipeline's KMS/secret store only; host pins the public key; manifest carries `keyId` (K6, N7-ready).

**Production→agent feedback channel:** host tags every Sentry event with `module.id` + `module.version` via error-boundary attribution (H11). A scheduled worker (every 6 h) queries the Sentry API, joins lightweight usage counters (emitted through `sdk.events`, aggregated by the same worker), and materializes `feedback/<moduleId>.json` in the bucket. Agents read a static JSON file — no Sentry credentials in any sandbox.

## 6. Verification & Promotion Pipeline

All stages emit/append to one `verification-report.json` conforming to a published `report.schema.json`: `{ stage, verdict: "pass"|"fail"|"warn", findings: [{ code, severity, evidence: { file?, line?, screenshot?, video?, log? }, suggestedFix? }], artifacts: [] }` — produced by agents, consumed by the next agent, rendered for the human.

- **Stage 1 (P1–P4, blocking):** `modkit verify` in CI = `tsc` against published SDK types, ESLint boundary rules, Jest Tier 1, Metro build, manifest/capability/mount validation against the host's exposed sets, allowlist check, and a **side-effect probe**: the bundle is evaluated in `node:vm` with all externals replaced by recording `Proxy`s; any call, network attempt, or global mutation before `mount()` fails with `MOD_E031` (C4).
- **Stage 1.5 (P5):** boot simulator, install dev shell, load candidate via dev registry, replay every `flows/*.flow.json` with argent; capture screenshots + screen recording; error-boundary trips or failed steps are blocking findings with the screenshot attached as evidence.
- **Stage 2 (P6/P7):** **code-review agent** (Claude, checklist prompt: capability appropriateness, data handling, exfiltration patterns, design-system usage, store compliance per K3) and **behavioral agent** (replays flows, compares outcome against `feature-spec.yaml` acceptance criteria, may improvise extra probes). Both append findings to the same report.
- **Stage 3 (P8/P9):** the **Promote UI** — one page per candidate in the admin service rendering the report (verdict banner, findings table, screenshot strip, embedded video, spec diff). One button: *Promote to staging / production* → `POST /promote` → sign → upload artifact + sourcemap → update channel manifest. Q5: staging passage is **per-module owner's choice**, a toggle on the same page. Kill and rollback buttons live on the module's page (P10/P11).

## 7. Security & App-Store Compliance

Per spec, stated honestly: modules are semi-trusted output of the owner's own pipeline (K5); the runtime is **soft isolation** — error boundaries, watchdog, kill-switch, capability-scoped SDK object — and capability scoping is a build/review-time convention, not a security boundary (R3; documented verbatim in `SECURITY.md`). Defenses against registry/CDN compromise: TLS + SHA-256 + pinned-key Ed25519 (K4). JS-only bundles on the same legal footing as EAS Update/CodePush; no primary-purpose changes, enforced as an explicit Stage-2 checklist item (K1/K3); per-module remote kill as the store-safety backstop (K2). No native code OTA ever (N10). Q3 (data lifecycle): on **kill**, module-namespaced storage is retained 30 days then purged; on **rollback**, retained (modules must treat storage as forward-compatible within a major version).

## 8. Versioning / Compatibility Contract (machine-checkable)

- Host declares `sdkVersion` (semver); module declares `requiredSdkVersion` range; host refuses on mismatch (H3) — checked at load *and* at publish against the live fleet's host versions.
- `@app/module-sdk` CI runs **api-extractor**; an unacknowledged API-report diff fails the host build, so SDK semver discipline is mechanical, not cultural (C7). Additive = minor; breaking = major, rare and batched.
- `modkit check --compat --json` extracts the module's actually-used SDK surface (ts-morph over typecheck results), diffs it against each published `sdk-api.<version>.json`, and **computes the widest correct `requiredSdkVersion` range**, emitting a fix-it if the manifest is too narrow or too generous. Agents never guess the range.
- Contract shape (ids, capabilities, mounts, SDK injection) is app-agnostic so later framework extraction (G6/C8) and a future hard-isolation transport swap (R4) don't break published modules.

## 9. Roadmap

- **Phase 1 — MVP loop (≈5 weeks, matches §7.5):** loader (fetch/hash/sig/eval/mount, error boundaries, kill-switch, cache), dev shell with hot module swap + `__DEVSHELL__` hooks, `create-module` scaffold, `modkit test|dev|check|build`, local + S3 registry with static manifests, manual `modkit publish` + promote CLI, Ed25519 signing. Exit: one real module developed by an agent end-to-end and killed/rolled back remotely.
- **Phase 2 — Pipeline & promote (≈4 weeks):** GitHub Actions Stage 1/1.5 with argent smoke flows, `report.schema.json`, AI code-review + behavioral agents, Promote UI with one-click promote/kill/rollback, staging channel, error-code taxonomy + ESLint fix-its complete.
- **Phase 3 — Feedback & scale (≈4 weeks):** Sentry tagging → `feedback/*.json` worker, `modkit feedback`, maintenance-agent playbook, `--compat` range computation, sourcemap symbolication, prefetch hints, embedded baseline modules (H8). Defer beyond v1: cohorts/percent rollout (N6), key rotation (N7), dashboards (N8), hard isolation (N1, trigger per Q7 = first non-pipeline module author).

## 10. Key Risks and Mitigations

| Risk | Mitigation |
|---|---|
| **Mock/real fidelity gap** — agents overfit to `@app/sdk-mock`, pass Tier 1, fail in Hermes. | Mock is generated from the same SDK type surface; contract tests run identical scenario fixtures against mock *and* dev shell nightly; cheap Tier 2 (<3 s) keeps agents honest. |
| **Hermes `eval` cost** on large bundles (no bytecode, N5). | Lazy evaluation (H10), size budget enforced in Stage 1 (warn 300 KB / fail 1 MB), bytecode is the designed Phase-4 optimization with manifest already carrying format field. |
| **Metro externals fragility** across RN/Expo upgrades. | Externals shim list is generated from one source of truth in host CI; dev-shell + SDK types are republished atomically per SDK version; `modkit shell install` always matches. |
| **Error-code taxonomy rot** — codes/fix-its drift from reality and mislead agents. | Codes live in one package (`@app/modkit-errors`) consumed by ESLint, bundler, loader, and docs; CI test asserts every thrown code has a docs entry and a fix-it or next step. |
| **Soft-isolation leakage** — a module corrupts shared state (R3 accepted). | Namespaced storage/state writes, Stage-2 review checklist item, per-module kill + crash tagging make blast radius detectable and reversible within one foreground cycle. |
| **Human gate rubber-stamping** — reports get approved unread. | Promote UI leads with behavior video + red/amber findings, blocks one-click when any `severity: "blocker"` finding exists; kill-switch keeps the cost of a wrong approval bounded (K2). |
