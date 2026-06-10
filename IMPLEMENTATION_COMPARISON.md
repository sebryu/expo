# Implementation Proposals — Pros & Cons Comparison

Six alternative implementation proposals were produced by independent agents.
**Set A** was designed from `GENERIC_PLAN.md` (the raw idea) — the alternatives differ by *technology strategy*.
**Set B** was designed from `REVIEWED_SPEC.md` (the interviewed, decision-logged spec) — all three honor the spec's decisions (soft isolation in v1, plain-JS modules, leaf modules, one-click promote) and differ by *priority axis*.

| # | File | Angle |
|---|------|-------|
| A1 | `IMPL_GENERIC_1_EXPO_NATIVE.md` | Bend expo-updates / EAS Update into a module-delivery system |
| A2 | `IMPL_GENERIC_2_MODULE_FEDERATION.md` | Re.Pack 5 + Module Federation v2 remote containers |
| A3 | `IMPL_GENERIC_3_SANDBOXED_RUNTIME.md` | One isolated JS engine per module + capability broker + remote-UI |
| B1 | `IMPL_REVIEWED_1_PRAGMATIC_MVP.md` | Fastest credible end-to-end MVP, zero servers |
| B2 | `IMPL_REVIEWED_2_PRODUCTION_GRADE.md` | Operational safety: containment, signing, health-gated rollout, runbooks |
| B3 | `IMPL_REVIEWED_3_AGENT_DX.md` | AI-agent iteration loop as the primary design driver |

---

## Set A — from GENERIC_PLAN

### A1. Expo-Native (`IMPL_GENERIC_1_EXPO_NATIVE.md`)

Each module is an EAS Update on its own branch (`module/<id>/<channel>`); a sidecar fork of expo-updates (`expo-module-host`) reuses its FileDownloader, SQLite cache, SelectionPolicy, and RSA code-signing stack to fetch/verify N module bundles. Modules are plain-JS Metro bundles with host deps externalized, evaluated into the shared Hermes runtime with a frozen capability-scoped SDK, mounted via an expo-router catch-all route. Only custom backend: a ~200-line composition/kill-switch Worker.

**Pros**
- Reuses battle-tested, shipped infrastructure (downloads, hashing, code signing, cache, rollout/rollback) — dramatically less custom code to build and operate.
- Mechanical host/module compatibility for free via expo-updates `runtimeVersion`; incompatible modules are never even downloaded.
- Excellent dev/prod parity: sandbox uses the exact same loader, SDK, manifest schema, and capability enforcement as production.
- Fast time-to-first-demo: Phase 1 needs no native code (open expo-updates protocol fetched from JS) — end-to-end loop provable in weeks.

**Cons**
- The sidecar fork of expo-updates internals is a permanent maintenance tax tracking upstream on every Expo SDK upgrade; multi-update usage is outside its tested design envelope.
- Soft isolation only: a pathological module (infinite loop, global tampering) is mitigated by review/signing, not technically contained.
- Plain-JS modules mean runtime parse cost on mount and larger artifacts than Hermes bytecode; low-end Android will feel it.
- Deep coupling to EAS conventions (branches-as-registry, channels, rollout tooling) and an unofficial Metro externals hack — vendor/ToS drift or Metro changes can break the pipeline.

### A2. Module Federation (`IMPL_GENERIC_2_MODULE_FEDERATION.md`)

Re.Pack 5 (Rspack) host shell + independently built Module Federation v2 remote containers (`.fmod` artifacts) downloaded by Re.Pack's ScriptManager with sha512 + ed25519 verification. Shared deps (react, react-native, expo-*) are eager host singletons; remotes are consume-only, so module bundles carry zero framework code. Registry: Postgres + S3/CDN with channels, targeting, health-gated rollout.

**Pros**
- True build isolation: modules are independently built/deployed artifacts in an industry-standard container format; a module build can never break the host build.
- Battle-tested machinery: ScriptManager + MF v2 runtime already solve chunk resolution, caching, signature hooks, and shared-singleton negotiation — far less bespoke loader code.
- Sandbox-equals-production by construction: the agent dev loop uses the exact ScriptManager/share-scope path as prod, just with a localhost resolver.
- Tiny module payloads, and `requiredVersion` failures reject the load gracefully (error boundary) instead of crashing with dual-React.

**Cons**
- Abandons the Metro-centric Expo toolchain: no `npx expo start`, expo-router incompatible (Metro `require.context`), expo-updates dropped — every Expo SDK upgrade carries Rspack-compat risk.
- Remote containers are interpreted plain JS on Hermes; large modules pay real parse/eval cost until a custom native HBC loader exists.
- Hard dependency on Callstack's Re.Pack / MF-runtime maintenance cadence tracking React Native releases.
- Security isolation is soft: shared Hermes runtime; capability scoping is convention + CI + signing, not a hard boundary.

### A3. Sandboxed Runtime (`IMPL_GENERIC_3_SANDBOXED_RUNTIME.md`)

Every module executes in its own JS engine (dedicated Hermes runtime via JSI, or QuickJS for untrusted code) on its own thread with heap caps and CPU watchdogs. A CapabilityBroker enforces deny-by-default, manifest-declared, signed capability grants with per-call audit logging. UI renders through a remote-UI protocol (react-reconciler emitting batched JSON ops; host materializes allow-listed design-system components). Signed `.emod` artifacts; dev sandbox is literally the prod sandbox.

**Pros**
- Strongest safety/security story: isolation enforced by the engine (separate heap, thread, watchdog) — a malicious or broken module provably cannot crash or corrupt the host; kill-switch = destroy a runtime.
- "AI agents can't break the core" is a structural guarantee, not a process; dev iteration is fast (respawn runtime, no Metro).
- The capability manifest is a mechanical, auditable review surface (capability diffs, per-call logs, rate limits) — ideal for eventual third-party modules and store-review escalations.
- No Metro multi-bundle hacks: modules are plain rollup bundles with zero react-native imports — sidesteps the bundler-reality risk entirely.

**Cons**
- Significant performance tax: 2–8 MB per Hermes instance plus thread-hop/JSON serialization on every interaction; chatty modules feel sluggish without careful batching.
- The remote-UI bridge is the hardest, riskiest engineering: gestures, animations, 60fps lists don't naturally cross a serialized boundary; module expressiveness is permanently capped by the host's component vocabulary.
- Highest build cost: custom native package, hermesc bytecode version matrix, custom registry, reconciler protocol — roughly 5 months to first production module.
- Multi-instance Hermes via JSI is off the beaten path; RN/Expo upgrades can break assumptions; no out-of-box DevTools across the sandbox boundary.

---

## Set B — from REVIEWED_SPEC

### B1. Pragmatic MVP (`IMPL_REVIEWED_1_PRAGMATIC_MVP.md`)

A ~500-line custom loader evaluating esbuild-built single-file JS bundles via `new Function` in shared Hermes, host singletons via a `__hostRequire__` global. No backend service: the registry is a git repo synced to Cloudflare R2 static JSON. One-click promote = merging an auto-generated promotion PR embedding AI review reports and simulator recordings; publish/kill/rollback via a `modctl` CLI + GitHub Actions. ed25519 signing with the pubkey pinned in the binary. Target: first AI-built module on device in ~3 weeks.

**Pros**
- Smallest credible surface area: one loader package, one CLI, two repos, zero servers — every part boring and replaceable without touching the module contract.
- Git-as-registry gives audit log, review UI, rollback history, and one-click promote for free via PR merge — no admin UI or API to build.
- The full production loading path (fetch→verify→eval→mount) is exercised from week 2, so contract bugs surface before any AI workflow exists.
- Explicit deferral table with concrete triggers and fallbacks makes scope cuts defensible rather than hand-wavy.

**Cons**
- Hinges on Hermes `eval`/`Function` of esbuild output behaving on device; mitigated by a day-1 spike, but the fallback (Metro multi-bundle) costs most of a week.
- esbuild instead of Metro stretches the spec's "Metro-compatible" wording; a reviewer could force a rebuild of the template toolchain.
- No service in front of R2: manifest unsigned in MVP, and cohort/percentage rollout requires adding a Worker later — two known retrofits.
- Self-hosted Mac runner for simulator smoke tests is a single point of failure and a security-sensitive box (runs candidate AI code); first thing to break at modules/day cadence.

### B2. Production-Grade (`IMPL_REVIEWED_2_PRODUCTION_GRADE.md`)

Soft isolation pushed to its hardest credible form: deep-frozen Proxy SDK facades, native JS-thread watchdog, MMKV-persisted crash-loop breakers, per-module memory/error budgets, eval-time global-tamper tripwire — framed honestly as bug containment, not a security boundary. Single-file Metro bundles (custom serializer) evaluated via `globalEvalWithSourceUrl` so `module://` stack frames give per-module Sentry attribution. S3/CloudFront + Lambda/DynamoDB control plane signing manifests; health-gated percentage rollout with automatic halt; two ed25519 KMS keys, rotation procedure, five disaster runbooks; phase-3 hard isolation = swapping the SdkBroker backend to RPC without touching the module contract.

**Pros**
- Strongest failure story of Set B: crash-loop breaker + watchdog + signed manifests + auto-halt mean a bad module self-quarantines and a compromised CDN cannot un-kill modules.
- The Proxy SDK facade doubles as the phase-3 seam: hard isolation becomes a loader/transport swap behind the same typed interface — an engineered, not hand-waved, migration path.
- `module://` source-URL crash attribution gives per-module observability essentially for free, covering async errors that error boundaries miss.
- Operational artifacts (runbooks, audit log, one-command kill/rollback) make single-owner operation realistic at a several-modules/day cadence.

**Cons**
- Most expensive of Set B: ~+40% over a minimal MVP (Phase 1 is 7–9 weeks); the rollout engine/console exceeds the spec's MVP slice — justified only if the cadence goal materializes.
- Custom Metro serializer is bespoke, Expo-SDK-upgrade-fragile plumbing with no community support.
- Several hardening mechanisms are detection-only and approximate (delta-based heap budgets, non-preemptive watchdog, sampled tripwire) — complexity and false-positive risk for limited enforcement.
- Operational surface (KMS, rollout gates, health webhooks) may overwhelm a single owner despite CLI/console mitigation; auto-halt thresholds need real traffic to tune.

### B3. Agent-DX First (`IMPL_REVIEWED_3_AGENT_DX.md`)

Treats the AI agent's edit→observe→fix loop as the bottleneck; runtime and delivery stay deliberately boring (single Metro bundle with externals shims, `globalEvalWithSourceUrl`, JSON manifests on R2, ed25519, tiny Hono admin API). Innovation budget goes to: a `modkit` CLI where every command emits `--json` with stable `MOD_Exxx` error codes and fix-its; a self-teaching scaffold (typed `defineModule` SDK generic over declared capabilities, generated `CONTRACT.md`/`AGENTS.md`); <3s hot-swap into a prebuilt dev shell; argent-driven recorded flows reused as pipeline smoke tests; one shared `report.schema.json` flowing from verification through AI review to a one-click promote UI; production Sentry→feedback JSON briefing maintenance agents.

**Pros**
- Directly optimizes the system's actual throughput constraint (agent cycle time and feedback signal quality), which compounds across every module shipped.
- Capability scoping enforced at the type level plus ESLint fix-its surfaces most contract violations at edit time, before the pipeline ever runs.
- Recorded simulator flows triple-serve as smoke tests, behavioral-review inputs, and human-gate evidence; the shared report schema makes every stage's output the next agent's input.
- Runtime simplicity (one bundler, eval loader, static JSON manifests) minimizes bundler risk and keeps the path to hard isolation open.

**Cons**
- Heavy bespoke tooling surface (modkit, error taxonomy, sdk-mock, dev-shell hooks, report schema) is a real maintenance tax for a single-owner project — it exceeds the runtime code in size.
- Mock-fidelity risk: the generated SDK mock can drift from real dev-shell behavior; agents over-optimizing for the fast tier ship violations that surface late.
- Hermes source-eval of plain-JS bundles makes first-mount latency a genuine concern for larger modules.
- The Metro externals-shim technique is sensitive to RN/Expo internals; every SDK/host upgrade requires regenerating shims, dev shell, and types in lockstep.

---

## Cross-Cutting Comparison

| Dimension | A1 Expo-native | A2 Federation | A3 Sandboxed | B1 MVP | B2 Production | B3 Agent-DX |
|---|---|---|---|---|---|---|
| Containment strength | Soft | Soft | **Hard** | Soft | Soft+ (hardened) | Soft |
| Time to first shipped module | Weeks | ~1–2 months | ~5 months | **~3 weeks** | 7–9 weeks | ~4–6 weeks |
| Custom code/infra to maintain | Low–med (fork) | Medium | **Highest** | **Lowest** | High | High (tooling) |
| Keeps Expo/Metro toolchain | **Yes** | **No** | Mostly (host) | Mostly (esbuild for modules) | Yes (custom serializer) | Yes (shims) |
| Module expressiveness | Full React | Full React | **Capped (remote-UI)** | Full React | Full React | Full React |
| Vendor/upstream coupling | EAS + fork | Callstack/Re.Pack | Low | Low | Low–med (AWS) | Low–med |
| Path to hard isolation | Weak | Weak | **Is the destination** | Open | **Engineered seam** | Open |
| AI-agent iteration loop | Good | Good | Fast but constrained | Good | Good | **Best** |
| Ops story (rollout, health, runbooks) | EAS-provided | Designed | Designed | Minimal, retrofit later | **Best** | Basic + feedback loop |

## Observations & Recommendation

1. **The two sets answer different questions.** Set A answers *"which loading technology?"* (EAS reuse vs. federation vs. engine isolation). Set B takes the spec's technology decisions as settled and answers *"what do we build first and harden when?"* — which is the more actionable question now that REVIEWED_SPEC exists. The interview demonstrably narrowed the design: all of Set B converged on the same runtime shape (single shared Hermes, single-file plain-JS bundle, eval with source URL, scoped SDK object, static JSON manifests, ed25519) and differs only in emphasis — that convergence is itself evidence the spec's decisions are coherent.

2. **A3 (hard sandbox) is the only proposal that fully delivers the original "AI agents literally cannot break the app" promise — and it's the one the reviewed spec deliberately deferred.** Its cost (remote-UI bridge, ~5 months, capped expressiveness) is justified only if third-party/untrusted module authors become a goal. Keep it as the documented phase-3 destination; B2's RPC-seam design shows how to get there without a contract rewrite.

3. **Recommended composite path:** start with **B1 (Pragmatic MVP)** to prove the end-to-end loop in ~3 weeks; adopt **B3's tooling roadmap** (modkit-style structured errors, typed scaffold, report schema) as the second investment, because agent cycle time is the real throughput constraint; pull in **B2's hardening** (signed manifests via a control plane, crash-loop breaker, health-gated rollout) when shipping cadence actually rises. If you'd rather not own a loader at all and accept EAS coupling, **A1** is the strongest alternative starting point; **A2** only wins if you're prepared to leave the Metro/Expo toolchain, which conflicts with building on Expo in the first place.

4. **Shared risks to spike early regardless of choice:** (a) Hermes eval of an independently-built bundle on real devices (day-1 spike in B1); (b) the host-singleton externalization technique (Metro hack / esbuild shims / MF share-scope — every soft-isolation proposal needs one); (c) the simulator-smoke-test runner as a security-sensitive choke point once AI-authored code flows through it.
