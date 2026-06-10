# REVIEWED_SPEC — Expo Host App with Remote Module Injection and AI-Agent Sandbox Development

Status: Reviewed specification (supersedes `GENERIC_PLAN.md`)
Normative language: **MUST** / **SHOULD** / **MAY** per RFC 2119 intent.

---

## 1. Overview

This project builds a single Expo / React Native application ("the product") composed of:

1. A **host app** (embedded core) shipped through the public app stores: navigation shell, auth/session, design system, shared services, all native code, and a **module loader**.
2. **Remote feature modules**: JS-only, self-contained feature units developed *outside* the core repository, published to a thin registry, downloaded at runtime, and mounted into the host at host-defined mount points.

Around this runtime model sits a **development model for AI agents**: each feature is developed by an agent inside an isolated workspace (the "sandbox") with no write access to the host repository. The unit of AI work equals the unit of shipping, verification, and rollback: **one module**. A fully automated verification pipeline (static checks, simulator smoke tests, AI review agents) runs before a one-click human promote gate publishes the module to users.

The driving rationale (Decision D1) is **containment and independent rollback of AI-written code**, not OTA delivery convenience. Whole-bundle OTA + feature flags was explicitly rejected because it merges AI code into the shared bundle and makes rollback global. Module-level loading is a **requirement**; implementations MAY realize it on top of existing OTA machinery (e.g., EAS Update) only if per-module containment and per-module independent rollback genuinely hold.

v1 targets **one app, single-tenant**. The module contract and SDK surface MUST nevertheless be designed for later extraction into a reusable framework (explicit long-term goal).

---

## 2. Goals and Non-Goals

### Goals (v1)

- G1. Load, mount, update, kill, and roll back individual feature modules at runtime without an app-store release and without affecting other modules or the host.
- G2. Give AI agents an isolated, high-velocity development environment in which nothing they do can modify or break the host app or core codebase.
- G3. A fully automated verification pipeline up to a one-click human promotion gate; human reviews an AI-produced report + recorded behavior, not (necessarily) code.
- G4. Ship to public App Store and Play Store in compliance with their interpreted-code policies.
- G5. Per-module remote kill-switch and instant per-module rollback from day one.
- G6. Design the module contract / SDK surface so it can later be extracted as a reusable framework without redesign.

### Non-Goals (v1) — explicitly deferred or out of scope

- N1. Hard runtime isolation (separate JS engine/worker per module). Documented later phase, required only if third-party/untrusted module authors ever become a goal.
- N2. Defense against malicious module *authors*. Modules are semi-trusted (see Threat Model, §9.2).
- N3. Module-to-module dependencies, direct imports between modules, or module-to-module navigation.
- N4. Multi-tenant registry / serving multiple apps.
- N5. Hermes bytecode module format (later optimization; accepts pinning modules to host Hermes version when adopted).
- N6. Cohort targeting and percentage-based staged rollout (post-MVP).
- N7. Signing key-rotation infrastructure (post-MVP; signing itself is in MVP).
- N8. Per-module observability dashboards (post-MVP; basic per-module crash *tagging* is in MVP).
- N9. Cloud container-per-agent sandbox hosting (later scaling step; v1 is local-first).
- N10. Shipping or hot-loading any native code OTA — permanently out of scope (store policy).

---

## 3. Target Users / Personas

| Persona | Role | Needs |
|---|---|---|
| **Owner** (single human) | Product owner, final approver, operator | One-click promote; trustworthy AI review reports + behavior recordings; kill-switch and rollback controls; minimal operational burden. |
| **AI feature agents** | Develop modules in sandboxes | Module template/scaffold; stable typed SDK; fast mocked-test loop; high-fidelity simulator loop; clear pass/fail feedback from the pipeline. |
| **AI review agents** | Verify candidate modules | Module artifact + manifest + feature spec + pipeline outputs; ability to drive the module in a simulator; structured report format. |
| **End users** | Use the shipped app | Stability (a broken module never takes down the app), offline behavior for cached modules, no awareness that features are modular. |
| **Future framework adopters** (long-term) | Other apps reusing the framework | Clean, documented, app-agnostic module contract. Not served in v1 beyond contract design discipline. |

---

## 4. System Architecture Requirements

Five components: **Host app**, **Modules**, **Registry backend**, **Sandbox**, **Verification & promotion pipeline**.

### 4.1 Host App (embedded core)

- H1. The host MUST contain: navigation shell, authentication/session, design system, shared services, **all** native code/modules, and the module loader + runtime.
- H2. The host MUST expose a **stable, versioned SDK surface** (the "bridge") as the only sanctioned way for modules to access app capabilities (navigation, storage, network, events, shared state, native features). The SDK MUST be handed to the module as a capability-scoped object at mount time; modules MUST NOT import host internals (enforced per S5/P3).
- H3. The host MUST declare an **SDK version** and refuse to load modules whose manifest requires an incompatible SDK version (explicit compatibility contract; semver-style: host accepts modules whose required SDK version is satisfied by the host's).
- H4. **Module loader** responsibilities: fetch the channel manifest from the registry; resolve which module versions apply to this host build; download, hash/signature-verify, cache, and evaluate module bundles; mount module entry points at host-defined mount points; unmount/kill on command.
- H5. **Containment:** every module mount point MUST be wrapped in an error boundary plus a load/evaluate watchdog. A module that throws at load or render MUST be unmounted and disabled locally (and reported), never crashing the host. Repeated failures MUST trip a local circuit breaker that stops retrying the module until a new version arrives.
- H6. **Kill-switch:** the host MUST honor a per-module remote kill flag from the registry manifest on every manifest refresh, and MUST hide/unmount killed modules within one app foreground cycle.
- H7. **Mount points:** the host defines (a) **route mounts** — named navigation routes a module can claim as whole screens, and (b) **extension slots** — explicit widget points the host exposes (e.g., a home-screen card list). No other injection into host-owned UI is possible.
- H8. The host MAY embed baseline copies of selected modules in the binary for first-run and offline behavior; embedded copies are superseded by newer registry versions once fetched.
- H9. Previously downloaded and verified modules MUST work offline from cache.
- H10. **Performance:** modules MUST be lazily evaluated (download/evaluate on first navigation to their mount point, or via host-directed prefetch). Loader overhead on cold start with zero modules SHOULD be negligible (<50 ms target on mid-range device).
- H11. **Observability (MVP scope):** all crash reports and error logs MUST be tagged with module id + version when the failure originates in a module (error-boundary attribution). Structured logging (key=value), not string concatenation.

### 4.2 Runtime Model (isolation level — decided)

- R1. v1 runs all modules in the **shared Hermes runtime** of the host ("soft isolation"). Modules register real React components/screens into the host's React tree.
- R2. Soft isolation mechanisms (all REQUIRED): per-mount error boundaries (H5), per-module kill-switch (H6), capability-scoped SDK object (H2), lint/CI-enforced import boundaries in the pipeline (P3), artifact hash + signature verification before evaluation (B4).
- R3. Capability scoping in v1 is a **convention enforced at build/review time**, not a runtime security boundary. This is accepted and MUST be stated in any security documentation.
- R4. The architecture SHOULD keep the module-facing API narrow enough that a later move to hard isolation (separate runtime/worker + RPC) changes the loader and SDK transport, not the module contract's declared semantics.

### 4.3 Modules

- M1. A module is a versioned artifact: one JS bundle + a declarative manifest + optional assets (images, fonts banned in v1 unless host-provided, JSON data allowed).
- M2. Modules are **JS-only**. Native capability access happens exclusively through SDK APIs backed by native code already shipped in the host.
- M3. Modules are **leaf nodes**: no module-to-module dependencies, imports, or direct navigation in v1. All cross-feature communication goes through host-mediated SDK services (events, shared-state APIs owned by the host).
- M4. A broken or incompatible module MUST degrade gracefully: the host hides its mount points and continues operating.

### 4.4 Registry Backend (default: thin custom registry)

- B1. **Default implementation:** object storage (e.g., S3) holding immutable artifacts + JSON manifests, fronted by a minimal API/CDN. This is the spec default, **not** a hard requirement: an implementation MAY bend EAS Update (or similar) to this job if it can prove per-module independence of publish/kill/rollback (D9).
- B2. **Channels:** at minimum `dev`, `staging`, `production`. Each channel manifest lists, per module: id, version, artifact URL, content hash, signature, kill flag, minimum/required host SDK version. The `dev` channel MAY point at sandbox/local builds.
- B3. Artifacts are immutable and content-addressed; "rollback" = repointing the channel manifest at a previous artifact (instant, per-module, no re-upload).
- B4. **Integrity:** every artifact MUST be hash-verified and signature-verified by the host before evaluation. Signing happens in the promotion pipeline; the host pins the verification public key in the binary. Key rotation infra is deferred (N7) but the manifest format MUST carry a key id so rotation can be added without format breakage.
- B5. Targeting in MVP is **per-channel only**. Per-user/cohort targeting and percentage staged rollout are deferred (N6) but the manifest schema SHOULD reserve fields for them.
- B6. Single-tenant: one app, one registry namespace (N4).

### 4.5 Sandbox (summary — full requirements in §6)

Isolated per-agent workspace, scaffolded from a module template, with no write access to the host repo; two test tiers (mocked harness, real dev-shell on simulator via the dev channel/local registry).

### 4.6 Pipeline (summary — full requirements in §7)

Automated gates → AI review agents → one-click human promote → sign → publish → channel rollout, with kill/rollback as first-class operations.

---

## 5. Module Contract Requirements

### 5.1 Manifest (declarative, validated by pipeline and host)

Each module ships a manifest containing at minimum:

- `id` (stable, globally unique within the app), `version` (semver), `entryPoint`
- `requiredSdkVersion` (semver range the host must satisfy)
- `capabilities`: explicit list of SDK capability groups requested (e.g., `navigation`, `storage.scoped`, `network`, `events`, `camera`). The host hands the module an SDK object containing **only** these (R2/R3 caveat applies).
- `mounts`: declared route mounts and/or extension-slot targets (must match mount points the host exposes; unknown mounts are a validation failure).
- Artifact metadata added at publish: content hash, signature, signing key id.

### 5.2 Dependencies

- C1. **Host-provided singletons (the only way to use these):** `react`, `react-native`, `expo-*` SDK packages, the design system, the navigation library, the state library. Modules reference them as externals; the loader resolves them from the host. Bundling a second copy is a pipeline validation failure.
- C2. Modules MAY bundle small **pure-JS** utility dependencies, subject to a **pipeline-enforced allowlist** (contents TBD — see Open Questions Q2). Anything with native bindings, dynamic code loading, or network side effects at import time is rejected.
- C3. **Format:** plain JS bundle (Metro-compatible, single file) in v1. Hermes bytecode is a deferred optimization (N5).

### 5.3 Runtime contract

- C4. The module's entry point exports a registration function/default export that receives the scoped SDK object and returns its mountable components for the declared mounts. No top-level side effects beyond pure definition (validated by smoke test: evaluating the bundle MUST be side-effect-free until mount).
- C5. Modules render real React components into the host tree (R1). They MUST use host design-system components for UI primitives where they exist.
- C6. **State & data:** module-local state is the module's own; durable storage and cross-feature shared state go through SDK storage/state APIs, namespaced by module id (so kill/rollback can identify, and optionally purge, a module's data).
- C7. **Versioning/skew:** compatibility is governed solely by `requiredSdkVersion` vs. the host's SDK version (H3). The SDK surface follows semver: additive changes minor-bump; breaking changes major-bump and are expected to be rare and batched.
- C8. The contract MUST be app-agnostic in shape (ids, capabilities, mounts, SDK injection) so extraction into a framework later does not break published modules (G6).

---

## 6. Sandbox Requirements for AI-Agent Development

- S1. **Local-first.** Each agent works in an isolated workspace: its own repository (or an isolated package with mechanically enforced boundaries) scaffolded from a **module template** containing the manifest skeleton, SDK type definitions, lint config with import-boundary rules, test harness, and build scripts.
- S2. **Core protection (the mechanical meaning of "cannot affect the core"):** agents have **no write access** to the host/core repository. The host is consumed only as a built artifact (dev-shell app) and a published SDK types package.
- S3. **Two test tiers:**
  - **Tier 1 — host-emulation harness:** mocked SDK implementation + Jest-level component/unit tests. Fast inner loop; runs entirely inside the workspace.
  - **Tier 2 — real dev shell:** the actual host app (dev build) running in a simulator/emulator, loading the agent's module from a local/dev registry **through the same loading path as production** (fetch → verify → evaluate → mount). High fidelity; this is the loop that catches contract violations the mock cannot.
- S4. The dev channel / local registry MUST accept unsigned or dev-signed artifacts so agents can iterate without the production signing step; production verification rules still apply on `staging`/`production`.
- S5. The workspace lint/typecheck config MUST enforce the import boundary (only SDK + allowlisted deps importable) so agents get violation feedback at edit time, not first at pipeline time.
- S6. Agents SHOULD have simulator automation tooling available in the sandbox to drive and observe Tier 2 themselves (see Assumptions A3 — owner uses argent MCP tooling).
- S7. Each feature starts from a **feature spec** (the input artifact for both the developing agent and, later, the AI behavioral reviewer — see P5). The template MUST include a place for it.
- S8. Cloud container-per-agent is a later scaling step (N9); nothing in the workspace design may assume a specific host machine beyond "can run Metro + a simulator".

---

## 7. Verification & Promotion Pipeline Requirements

Stages run in order; every stage before the human gate is fully automated (required by the target cadence: multiple modules/week initially, growing toward several/day).

### 7.1 Blocking automated checks (Stage 1)

- P1. Typecheck against the published SDK types.
- P2. Lint, including import-boundary rules (no host internals, only allowlisted bundled deps).
- P3. Unit tests (Tier 1 harness).
- P4. Build: produce the JS bundle; validate manifest schema; validate `capabilities` and `mounts` against the host's exposed sets; validate the bundled-dependency allowlist; verify no top-level side effects on evaluation.
- P5. **Simulator smoke test:** install the dev shell, load the candidate module via the dev registry, automatically drive the module's declared entry flows, and capture **screenshots/video artifacts**. Crashes, error-boundary trips, or failed flow steps are blocking.

### 7.2 AI review (Stage 2)

- P6. **Code review agent:** quality + security review of the module source (capability appropriateness, data handling, obvious injection/exfiltration patterns, dead code, design-system usage). Produces a structured report.
- P7. **Behavioral verification agent:** verifies recorded behavior (and MAY re-drive the simulator) against the feature spec (S7). Produces a pass/fail-with-findings report.

### 7.3 Human gate (Stage 3)

- P8. The owner approves from the AI review reports + recorded behavior artifacts; reading code is optional. Approval MUST be a **one-click promote**.
- P9. Promote = package → sign (production key) → publish artifact to registry → update target channel manifest. Promotion to `production` MAY pass through `staging` first (owner's choice per module).

### 7.4 Operations (continuous)

- P10. **Kill:** one-command/one-click per-module kill that flips the manifest kill flag (effective per H6). Required both operationally and as an app-review safety mechanism (Constraint K2).
- P11. **Rollback:** one-command/one-click repoint of a channel manifest to a previous artifact version, per module, with no rebuild (B3).
- P12. Crash reports tagged with module id + version (H11) feed back into kill/rollback decisions; automation of that feedback loop is post-MVP.

### 7.5 MVP slice (the smallest end-to-end build)

Dev shell + module loader fetching a static JSON manifest from a simple backend (S3 or local server) + manual promote CLI + per-module kill-switch + error-boundary containment + artifact hash/signature check. Deferred from MVP: cohort targeting, percentage rollout, key rotation, observability dashboards (crash *tagging* stays in).

---

## 8. Constraints

### 8.1 App-store compliance

- K1. Public App Store + Play Store distribution is the end state (TestFlight/internal during development). Remote code MUST therefore be interpreted JS only, MUST run in the framework the store guidelines sanction for OTA JS (the same legal footing as EAS Update/CodePush), and modules MUST NOT change the app's primary purpose or unlock store-significant functionality (payments circumvention, app-within-app marketplaces, etc.).
- K2. The per-module remote kill-switch (P10/H6) is REQUIRED partly as a review-safety mechanism: any module that endangers store compliance can be removed from all users without a release.
- K3. The feature-spec and AI code-review stages MUST include a compliance check item ("does this module change the app's primary purpose or violate store policy?").

### 8.2 Security

- K4. Transport security (TLS) + artifact content hash + signature verification (B4) protect against compromised registry/CDN. This — not runtime sandboxing — is the chosen defense for that threat (D3).
- K5. Modules are **semi-trusted** (produced by the owner's own pipeline). Runtime capability scoping is best-effort in v1 (R3). Documents and threat-model discussions MUST NOT claim runtime sandboxing strength v1 does not have.
- K6. Signing keys: production signing happens only in the promotion pipeline; the verification key is pinned in the host binary; manifest carries key id for future rotation (B4, N7).

### 8.3 Native ceiling

- K7. A module can only use native features the host already ships. Adding a native capability requires a host app-store release; the SDK and host release cadence MUST treat "new native capability" as the only event that forces a binary release.
- K8. Consequently the host SHOULD pre-embed a deliberately chosen set of native capabilities (camera, location, secure storage, etc., as the product roadmap demands) ahead of module needs.

### 8.4 Bundler reality

- K9. Metro assumes a single static bundle; the loader requires deliberate multi-bundle architecture (module bundles built with host-provided externals — C1). This is a known engineering cost accepted by Decision D1. Implementation options (custom Metro config, Re.Pack/module federation, bending EAS Update) are left to implementation proposals, constrained by B1.

---

## 9. Decision Log

| # | Interview question | Answer (owner/proxy) | Resulting decision |
|---|---|---|---|
| D1 | Why per-module injection vs whole-bundle OTA + flags? | (a) Unit of AI work must equal unit of shipping/verification/rollback; whole-bundle OTA poisons the shared bundle and makes rollback global. (b) Modules live outside the core repo, so agents never get write access to core. Deploy cadence is a side effect, not the driver. | Real module-level loading is a **requirement**. Proposals MAY build on OTA machinery only if per-module containment + independent rollback genuinely hold (B1, K9). |
| D2 | One app or reusable framework? | v1 is a vehicle for one app, single-tenant registry; framework extraction is an explicit long-term goal. | Build single-tenant; design the module contract/SDK as if extractable (G6, C8). Docs code-first for now. |
| D3 | Threat model for runtime isolation? | Primary adversary is buggy AI-generated code; modules semi-trusted; compromised registry handled by TLS + signing, not runtime sandboxing. | v1 = shared Hermes runtime, soft isolation (error boundaries, kill-switch, scoped SDK object, lint/CI import boundaries). Modules register real React screens. Hard isolation is a documented later phase (R1–R4, N1, N2). |
| D4 | Module contract: deps, format, mount granularity? | Host-provided singletons only for core libs; small pure-JS deps via pipeline allowlist; never native. Plain JS bundle v1 (bytecode deferred). Whole screens at host routes + explicit extension slots; no arbitrary injection. | C1–C3, H7, N5. |
| D5 | What is the AI sandbox concretely? | Local-first; per-agent isolated repo/package from a module template; no write access to host repo; two test tiers (mocked harness + real dev shell on simulator via local/dev registry, same loading path as prod). Cloud per-agent containers later. | §6 (S1–S8), N9. |
| D6 | Verification bar & human gate? | Blocking: typecheck, lint w/ boundaries, unit tests, build, manifest/capability validation, automated simulator smoke test with screenshots/video. Then AI code review + AI behavioral verification vs feature spec. Human approves from reports + recordings; one-click promote. Cadence: multiple/week → several/day. | §7 (P1–P9). Everything pre-human fully automated. |
| D7 | Distribution & compliance? | Public App Store + Play Store end state; TestFlight/internal during development. | JS-only modules, no primary-purpose changes, per-module remote kill-switch REQUIRED (K1–K3, P10). |
| D8 | Inter-module relationships in v1? | Strictly leaf modules at host-defined mount points; communication only via host-mediated SDK services; module-to-module deps/imports out of scope, revisit after several modules exist. | M3, N3. |
| D9 | MVP slice & build-vs-buy? | MVP: dev shell + loader + static JSON manifest on simple backend + manual promote CLI + kill-switch + error boundaries + hash/signature check. Deferred: cohorts, % rollout, key rotation, observability dashboards (crash tagging in). Lean to thin custom registry; EAS-based approach allowed as an explored alternative. | §7.5, B1, N6–N8. |

---

## 10. Explicit Assumptions

Proxy-extrapolated answers, marked ASSUMPTION by the orchestrator — to be confirmed by the owner:

- **A1 (from D3):** The owner prioritizes developer experience and shipping speed over hard security boundaries at this stage; soft isolation is acceptable until third-party modules become a goal.
- **A2 (from D4):** The pure-JS bundled-dependency **allowlist details** (which packages, size limits, audit process) are extrapolated; only the existence of a pipeline-enforced allowlist is owner-stated.
- **A3 (from D5):** Simulator automation tooling (the owner's argent MCP tooling) is available to agents inside the sandbox for Tier-2 testing and to the pipeline for smoke tests (P5).

Additional assumptions implicit in this spec (flagged by the reviewer, not the owner):

- **A4:** The host's state library and navigation library choices are stable enough to be frozen into the SDK singleton set before the first module ships.
- **A5:** One signing key (plus key-id field for future rotation) is acceptable operational risk for MVP.
- **A6:** "Several modules per day" cadence is aspirational sizing for pipeline automation, not a hard SLA.

---

## 11. Remaining Open Questions

- **Q1. SDK v1 surface enumeration.** The exact capability groups and APIs (navigation, storage, network, events, shared state, which native features) must be enumerated before the first module template ships. (GENERIC_PLAN open question 5 — narrowed but not closed.)
- **Q2. Bundled-dependency allowlist contents** and its governance (who adds packages, what audit). (A2.)
- **Q3. Module data lifecycle on kill/rollback:** is module-namespaced storage purged, retained, or versioned when a module is killed or rolled back? (C6 names the namespace; the lifecycle policy is undecided.)
- **Q4. Loader implementation choice:** custom Metro multi-bundle build vs. Re.Pack/module federation vs. EAS-Update-based realization — to be settled by implementation proposals under the B1/D1 constraints.
- **Q5. Staging discipline:** is `staging` channel passage mandatory before `production`, or per-module owner's choice (P9 currently says MAY)?
- **Q6. Feature-spec format** consumed by developing agents and behavioral reviewers (S7, P7): freeform markdown vs. structured (e.g., flows + acceptance criteria) — structured is recommended for automatable behavioral verification.
- **Q7. Hard-isolation trigger:** what concrete event (first third-party module? first external contributor?) flips N1 from deferred to required, and what migration path does R4 imply for then-published modules?
