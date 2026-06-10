# IMPL_GENERIC_3 — Sandboxed Runtime: One Isolated JS Engine per Module, Capability Bridge, Host-Rendered UI

Implements `GENERIC_PLAN.md`. Sibling proposals explore softer isolation; this one takes the hard-sandbox position.

## 1. Summary

Every remote module runs in its **own JS engine instance** (a dedicated Hermes runtime created via JSI, with QuickJS as a constrained fallback profile), on its own thread, with **no shared globals, no shared heap, and no access to React Native** — only a narrow, capability-gated message bridge to the host. Modules cannot render natively; instead they drive a **host-rendered component tree** through a remote-UI protocol (react-reconciler inside the sandbox emitting JSON mutation ops; the host materializes them using allow-listed design-system components). The exact same sandbox binary is the AI development environment: an agent iterating on a module is, by construction, incapable of touching host code, host state, or other modules. Delivery is a signed-artifact registry with channels, staged rollout, and instant rollback.

### What makes this approach distinct

- **Isolation is enforced by the engine, not by convention.** A module cannot crash, freeze, or corrupt the host even if it is malicious, because it has no pointer into the host heap — vs. sibling approaches where modules share the main runtime and rely on error boundaries and code review.
- **Dev sandbox ≡ prod sandbox.** "AI agents can't break the core" is a property of the runtime, not of the workflow.
- **Capability model is mechanical**: every host interaction is a bridge call that can be enumerated, audited, rate-limited, logged per module, and revoked at runtime (kill-switch = destroy the runtime).
- Trade-off accepted up front: per-module memory overhead and a UI protocol that limits modules to the host's component vocabulary.

## 2. Architecture overview

```
┌─────────────────────────────── Host App (Expo, store-shipped) ───────────────────────────────┐
│  Main Hermes runtime (React Native, untouched by modules)                                     │
│  ┌─────────────┐  ┌────────────────┐  ┌───────────────────────────────────────────────┐      │
│  │ Nav shell    │  │ SurfaceRenderer│  │ expo-sandbox-runtime (native, JSI)            │      │
│  │ Auth/session │  │ applies UI ops │  │  ┌─────────────┐ ┌─────────────┐ ┌──────────┐ │      │
│  │ Design system│←─│ ← op batches   │←─┤  │ Sandbox A   │ │ Sandbox B   │ │ Sandbox C│ │      │
│  │ ModuleLoader │  │ → events       │──┤  │ Hermes rt   │ │ Hermes rt   │ │ QuickJS  │ │      │
│  └─────────────┘  └────────────────┘  │  │ own thread  │ │ own thread  │ │ own thrd │ │      │
│         │                             │  │ heap cap    │ │ heap cap    │ │ gas meter│ │      │
│  ┌──────▼───────────────────────┐     │  └──────┬──────┘ └──────┬──────┘ └────┬─────┘ │      │
│  │ CapabilityBroker             │◄────┴─────────┴───────────────┴─────────────┘       │      │
│  │ grants ← signed manifest     │   single choke point: __host_call(cap, method, json)│      │
│  │ nav / kv / fetch / surface…  │  └───────────────────────────────────────────────┘  │      │
│  └──────────────────────────────┘                                                     │      │
└───────────────▲───────────────────────────────────────────────────────────────────────┘      │
                │ HTTPS: manifest poll, .emod download, ed25519 verify, cache                   │
        ┌───────┴────────┐      ┌──────────────┐      ┌───────────────────────────┐
        │ Module Registry │◄────│ Promotion CI │◄────│ AI dev sandbox (same       │
        │ channels/rollout│      │ + AI review  │      │ expo-sandbox-runtime in a │
        │ artifacts + sigs│      │ + human gate │      │ "Sandbox Host" dev app)   │
        └────────────────┘      └──────────────┘      └───────────────────────────┘
```

Components:
- **`expo-sandbox-runtime`** (new native package, iOS/Android): creates per-module `facebook::hermes::makeHermesRuntime()` instances via JSI, each on a dedicated thread with `RuntimeConfig` heap caps and `HermesRuntime::watchTimeLimit()` watchdogs. Exposes exactly two host functions into each sandbox global: `__host_call(capability, method, argsJson) -> Promise<json>` and `__host_subscribe(topic, cbId)`. Nothing else — no `fetch`, no `require`, no Turbo Modules. A QuickJS profile (via `react-native-quickjs`, `JS_SetMemoryLimit` + interrupt-handler gas metering) exists for modules flagged "untrusted/strict".
- **CapabilityBroker** (host JS + native): maps `(moduleId, capability)` to granted implementations; deny-by-default; structured audit log (`module=`, `cap=`, `method=`, `latency=`) per call.
- **SurfaceRenderer** (host JS): a React component `<ModuleSurface moduleId="...">` that applies UI mutation ops from the sandbox to a host-side tree of allow-listed components (adapted from Shopify's `@remote-ui/core` receiver, the proven prior art from Checkout Extensions).
- **ModuleLoader**: polls registry manifest per channel/user, downloads `.emod` artifacts, verifies ed25519 signatures natively (CryptoKit / Tink) before any byte reaches a runtime, caches in app storage for offline.

## 3. Module format & loading

**Artifact `.emod`** — a zip:
```
manifest.json          # signed canonical JSON (RFC 8785)
manifest.sig           # ed25519 over manifest.json
bundle.hbc.96          # Hermes bytecode, keyed by HBC version
bundle.js              # plain-JS fallback (Hermes evaluates source if HBC version mismatches)
assets/                # images etc., served to host components by URI handle
```

**Manifest schema** (the unit of review and signing):
```json
{
  "id": "com.acme.expense-tracker",
  "version": "1.4.0",
  "bridgeApi": ">=1.2 <2",
  "runtimeProfile": "hermes",            // or "quickjs-strict"
  "entry": "bundle",
  "capabilities": [
    { "cap": "surface.render", "surfaces": ["tab:expenses", "screen:expenses/*"] },
    { "cap": "storage.kv", "quotaBytes": 1048576 },
    { "cap": "net.fetch", "hosts": ["api.acme.com"], "maxRps": 5 },
    { "cap": "nav.push", "routes": ["expenses/*"] }
  ],
  "limits": { "heapBytes": 33554432, "cpuMsPerTask": 50 },
  "artifacts": { "bundle.hbc.96": "sha512-...", "bundle.js": "sha512-..." }
}
```

**Build**: module workspace is plain TS + React; bundled with `rollup` (no Metro needed — modules never import react-native) into a single IIFE, `react` + `@remote-ui/react` compiled in or provided as sandbox-side prelude; then `hermesc -emit-binary` using the hermesc matching each supported host Hermes version. CI emits one `.emod` with HBC for every active host bytecode version plus the JS fallback.

**Loading sequence**: verify signature → check `bridgeApi` against host's advertised version → spawn runtime with manifest `limits` → evaluate prelude (Promise/queueMicrotask shims, bridge SDK `@yourco/module-sdk` which wraps `__host_call` in typed APIs) → evaluate bundle → module default-exports `{ onMount(surface), onUnmount() }`.

**UI rendering**: inside the sandbox, the module renders normal JSX through a `react-reconciler` custom renderer (per `@remote-ui/react`). The reconciler serializes mutations — `CREATE type props`, `APPEND`, `UPDATE_PROPS`, `REMOVE`, with props restricted to JSON + callback ids — batched per 16ms frame into one bridge message. Host's SurfaceRenderer applies them to real components from an explicit registry (`Stack`, `Text`, `Button`, `TextField`, `List` …). Events serialize back as `(cbId, payload)`. Lists virtualize host-side: module supplies windowed data via a `List` data-source protocol; FlashList on the host requests pages.

## 4. Sandbox strategy for AI-agent development

The development environment is the production isolation, not an emulation of it:

1. Agent gets a **module workspace** (template repo): `src/`, `manifest.json`, `module-sdk` types, vitest + a Node-side bridge simulator for unit tests. The workspace contains zero host source.
2. `modkit dev` starts a watcher that rebuilds the bundle and serves it on a local port.
3. The **Sandbox Host** — a dev build of the real host shell with the dev channel pointed at `localhost` — loads the bundle into the *same* `expo-sandbox-runtime` sandbox used in production, with the manifest's declared capabilities and limits enforced identically. Capability violations fail loudly in dev exactly as in prod.
4. The agent iterates: edit → auto-reload (destroy runtime, respawn, re-evaluate — sub-second, no Metro involvement) → drive the UI in the simulator via MCP tools (Argent) → read the per-module structured log stream exposed by `modkit logs`.
5. Containment is structural: the agent's write access is the module workspace; its runtime blast radius is one heap-capped, time-limited engine. It cannot import host code (none is present), cannot reach undeclared hosts (broker blocks), cannot exceed quota. A crashed or spinning module is killed by the watchdog and the Sandbox Host stays up — the agent observes the failure in logs and keeps iterating.
6. `modkit verify` runs locally the same checks CI runs (Section 6), so agents converge before promotion.

## 5. Backend / registry design

Small custom service (Fastify + Postgres + S3-compatible blob store; or Cloudflare Workers + R2 + D1). EAS Update is not reused because its unit is "whole app bundle", not "one module among many".

- **Tables**: `modules(id, owner, public_keys[])`, `versions(module_id, semver, manifest_json, artifact_urls, status)`, `channels(app_id, name)`, `releases(channel, module_id, version, rollout_pct, constraints{hostVersion, bridgeApi, hbcVersions})`, `kill_switches(module_id, reason)`.
- **Client API**: `GET /v1/apps/:app/channels/:ch/manifest?host=1.8.0&bridge=1.3&hbc=96&user=<hash>` → list of module manifests + artifact URLs resolved for that host (HBC variant selection, rollout bucketing by stable user hash). Responses signed; artifacts content-addressed and CDN-cached.
- **Channels**: `dev` (sandbox builds, auto-publish from `modkit`), `staging`, `production`. Per-user/cohort targeting via release constraints.
- **Capability review is registry-enforced**: a version whose manifest requests capabilities beyond the module's previously approved grant set is held in `status=pending_capability_review` and cannot be released to staging/production until a human approves the diff. Capability grants, not code, are the primary review surface.
- **Rollback**: `releases` is append-only pointers; rollback = repoint + push kill-switch flag that clients poll (and receive via silent push), causing immediate runtime teardown + cache eviction.

## 6. Verification & promotion pipeline

GitHub Actions per module repo:
1. **Static**: `tsc --noEmit` against `module-sdk` types; ESLint with a sandbox ruleset (bans `eval`, `Function`, dynamic `__host_call` capability strings); manifest schema validation; dependency audit (modules may only depend on an allow-listed npm set — `react`, `zod`, etc.).
2. **Unit**: vitest against the Node bridge simulator (capability mocks with the same quota/deny semantics).
3. **Runtime smoke**: boot Sandbox Host in CI simulator/emulator, load the artifact, run scripted flows (Maestro, or Argent flows recorded during development); assert no watchdog kills, no capability denials, memory under cap, op-throughput under budget.
4. **AI review agents** (Claude Agent SDK): (a) behavior agent drives the module in the simulator against the feature spec and writes a verdict; (b) code agent reviews the diff for capability creep, data exfiltration patterns (encoding user data into allowed-host URLs), and quality. Both verdicts attach to the version.
5. **Human gate**: owner sees spec, AI verdicts, capability diff, and a video of the smoke run; one click signs the manifest with the release key (HSM/KMS-held; CI never holds it) and publishes to `staging`.
6. **Staged rollout**: production release starts at 1% → 10% → 50% → 100%, auto-halted if per-module crash/watchdog/denial metrics (Section 7 observability) exceed thresholds. Rollback as in Section 5.

## 7. Security & app-store compliance

This architecture's strongest suit — the security claims are checkable:

- **No ambient authority**: the sandbox global contains the language builtins plus two bridge functions. No network, filesystem, timers-to-native, or reflection into the host. Every capability is (1) declared in the signed manifest, (2) approved in registry review, (3) enforced at the broker on every call, (4) logged.
- **Capability semantics**: `storage.kv` is namespaced `module/<id>/` with byte quota; `net.fetch` is host-allow-listed, TLS-only, rate-limited, with response-size caps; `nav.push` only to routes the module owns; `surface.render` only to declared surface slots; sensitive caps (`contacts.read`, `location`) additionally require a host-rendered (not module-rendered) user consent sheet, so a module cannot fake the prompt.
- **Resource containment**: per-runtime heap cap (Hermes `GCConfig::MaxHeapSize`), `watchTimeLimit` CPU watchdog per task, bridge message size/rate limits. Kill-switch destroys the runtime and unmounts the surface; host shows a fallback card.
- **Supply chain**: ed25519 manifest signatures verified natively before evaluation; artifact hashes pinned in the manifest; release key separate from dev keys; registry responses signed to prevent CDN tampering.
- **App-store position**: identical legal footing to EAS Update/CodePush — interpreted JS executed by the embedded engine, no native code OTA, modules extend rather than change the app's primary purpose (Apple Guideline 2.5.2 / Google Play "Device and Network Abuse" policy). The capability manifest is also a strong answer in review escalations: we can show exactly what downloaded code is able to do.

## 8. Versioning / compatibility contract

- **`bridgeApi` semver** is the single host↔module contract, covering: bridge call envelope, the typed capability method set, the UI component registry (component names + prop schemas, generated from host source into `@yourco/module-sdk`), and event payload shapes.
- Host advertises `bridgeApi` (e.g. `1.3`) plus supported HBC versions in its manifest request; registry returns only compatible versions. Host-side double-check refuses out-of-range modules and renders the fallback card (with embedded baseline modules as offline/first-run fallback).
- **Rules**: additive changes (new capability methods, new components, optional props) bump minor; removals/renames bump major; host supports majors N and N−1 for ≥6 months; component registry entries are never repurposed, only deprecated. Contract conformance is enforced by generated JSON-schema validation of bridge messages in dev/staging (sampled in prod).
- **Engine skew**: HBC variants per supported Hermes bytecode version; `bundle.js` source fallback covers gaps at a startup-time cost.

## 9. MVP roadmap

**Phase 1 — Sandbox core (≈6–8 weeks, 2 eng)**: `expo-sandbox-runtime` (iOS+Android, Hermes-only: spawn/destroy, thread, heap cap, watchdog, `__host_call`); CapabilityBroker with `storage.kv`, `net.fetch`, `log`; remote-UI prelude + SurfaceRenderer with ~10 components; load one signed `.emod` from local disk into a demo host. Exit: a sandboxed to-do module renders and survives a deliberately hostile sibling module.

**Phase 2 — Delivery + dev loop (≈6 weeks, 2 eng)**: registry service (channels, releases, kill-switch, rollout bucketing); ModuleLoader with caching/offline; `modkit` CLI (dev server, verify, publish); Sandbox Host dev app; signing pipeline with KMS release key. Exit: an AI agent builds a module in the workspace and ships it to `staging` end-to-end without human file edits outside the workspace.

**Phase 3 — Verification + hardening (≈8 weeks, 2–3 eng)**: CI smoke harness, AI review agents, human gate UI, staged rollout + auto-halt metrics; QuickJS strict profile; `nav.*`/`surface` slot capabilities and List virtualization protocol; per-module observability (crash, watchdog, denial, bridge-latency dashboards). Exit: first production module at 100% rollout with a rehearsed rollback.

## 10. Key risks and mitigations

| Risk | Honest assessment | Mitigation |
|---|---|---|
| **Memory overhead per runtime** | Each Hermes instance costs roughly 2–8 MB baseline before module code; 10 modules ≠ free | Heap caps per manifest; lazy spawn on surface visibility; destroy on background + state snapshot via `storage.kv`; QuickJS profile (~1 MB) for small modules |
| **UI-bridge complexity & expressiveness** | The remote-UI protocol is the project's hardest engineering; gestures, animations, and 60fps lists don't naturally cross a serialized bridge | Adapt `@remote-ui/core` rather than invent; declarative animation/gesture props executed host-side (Reanimated under host components); host-side list virtualization; accept that some UI is host-only and say so in the SDK docs |
| **Bridge latency on chatty modules** | Every interaction is thread-hop + JSON; naive modules will feel sluggish | Per-frame op batching; binary envelope later if profiling demands; budget alerts in dev (`>N ops/frame` warning); design SDK APIs coarse-grained |
| **Hermes multi-runtime is off the beaten path** | `makeHermesRuntime` multi-instance works (worklets/Reanimated prove it) but RN upgrades can break assumptions; hermesc version matrix is real toil | Pin Hermes via Expo SDK cadence; CI builds HBC matrix automatically; JS-source fallback always shipped; isolate all engine code in `expo-sandbox-runtime` behind a small interface (QuickJS as second backend keeps us honest) |
| **Capability review becomes rubber-stamping** | Humans approve manifests, not code; a "fetch to api.acme.com" grant can still exfiltrate data the module legitimately holds | Capability *diffs* (not full lists) as the review unit; AI code-review agent specifically hunts exfiltration patterns; broker logs enable retroactive audit; sensitive data only reachable via host-mediated consent caps |
| **Debugging across the boundary** | No Chrome DevTools attached to sandbox runtimes out of the box | Hermes runtimes support CDP — Phase 3 exposes an inspector port per sandbox in dev builds; structured per-module logs from day one |
