# GENERIC_PLAN — Expo-Based Framework with Remote Module Injection and AI Sandbox Development

## Vision

Build a custom application framework on top of Expo / React Native in which an app is composed of:

1. **An embedded core ("host" / "shell")** — the stable part of the application, shipped through the app stores.
2. **Remote feature modules** — additional, self-contained pieces of the application downloaded at runtime from a backend and injected into the running app.

Around this runtime model, establish a **development model for AI agents**: new features are developed inside a **sandbox-like environment** where AI models can freely work and experiment **without any risk of breaking the core application**. When work is considered done — either by the owner's judgment or after verification by AI review agents — the feature is packaged and **shipped as a module** to the core application.

## Core Concepts

### 1. Host App (Embedded Core)

- Navigation shell, authentication/session, design system, and shared services.
- **Module loader + runtime**: discovers, downloads, validates, and mounts remote modules.
- A **stable SDK surface** (bridge) that modules use to access app capabilities (navigation, storage, network, native features). All native code lives here — modules are JS-only.
- Baseline modules may be embedded in the binary for first-run and offline behavior.

### 2. Remote Modules

- Self-contained feature units: JS/TS code, a declarative **manifest** (id, version, entry point, required SDK version, requested capabilities), and assets.
- Versioned, signed, and downloaded from the backend; loaded/injected at runtime.
- Must degrade gracefully: a broken or incompatible module must never take down the host.

### 3. Module Backend (Registry & Delivery)

- Stores module artifacts and versions.
- **Channels** (e.g. dev / staging / production) and targeting (per-user, per-cohort, staged rollout).
- Artifact signing and integrity verification; instant **rollback** to a previous module version.

### 4. Sandbox Development Environment (for AI Agents)

- AI agents develop modules in isolation from the core app — a workspace where they can run, test, and iterate on a module against a host emulation or a dev build of the shell.
- **Failure containment**: nothing an agent does in the sandbox can affect the production app or the core codebase.
- Agents interact with the app only through the stable SDK surface, mirroring the production module contract.

### 5. Verification & Promotion Pipeline

- Automated checks: type-checking, lint, unit tests, and runtime smoke tests (e.g. driving the module in a simulator).
- **AI review agents** verify behavior and code quality.
- Human approval gate (the owner decides "work is done").
- On approval: package → sign → publish to registry → staged rollout to the core application.

## Requirements

### Functional

- Load, mount, update, and unload modules at runtime without an app-store release.
- Module discovery driven by a backend-provided manifest (per user/channel).
- Side-by-side versions per channel; dev channel can point at sandbox builds.
- Modules can register screens/routes, components, and background logic via the SDK.

### Non-Functional

- **Safety**: a faulty module cannot crash or corrupt the host (error boundaries, watchdogs, kill-switch per module).
- **Security**: artifacts are signed and verified; modules get capability-scoped access only.
- **Compatibility**: explicit SDK version contract between host and modules; host refuses incompatible modules.
- **Performance**: lazy loading, caching of downloaded modules, minimal startup overhead.
- **Offline**: previously downloaded modules work offline; embedded fallbacks where needed.
- **Observability**: per-module crash reporting, logging, and usage metrics.

## Constraints & Risks

- **App store policy**: Apple and Google restrict downloadable executable code. Interpreted JS that does not change the app's primary purpose is generally permissible (this is how OTA updates like EAS Update / CodePush operate), but native code can never be shipped OTA — all native capabilities must be embedded in the host ahead of time.
- **Native dependency ceiling**: a module can only use native features the host already ships. Adding a new native capability requires an app-store release of the host.
- **Version skew**: many module versions in the wild against many host versions — needs a strict compatibility contract.
- **Security of remote code**: remote injection is an attack vector; requires signing, transport security, and runtime sandboxing/capability limits.
- **Bundler reality**: Metro assumes a single static bundle; multi-bundle / runtime-loading requires deliberate architecture (bundle splitting, module federation, or a separate runtime).

## Open Questions (to refine in review)

1. Module format: plain JS bundle, Hermes bytecode, or both?
2. Isolation level at runtime: same JS runtime with soft boundaries vs. separate engine/worker per module?
3. How exactly does the AI sandbox map to the runtime sandbox — same contract, same tooling?
4. Can modules depend on other modules? Shared state and navigation integration?
5. What is the minimal SDK/capability surface for v1?
6. Single app (personal/product) first, or a reusable framework for many apps from day one?
7. Verification bar: what must AI verifiers check before a human ever looks at it?
