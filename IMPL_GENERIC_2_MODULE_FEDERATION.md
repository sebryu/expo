# IMPL_GENERIC_2 — Module Federation Architecture (Re.Pack / MF v2)

Implementation proposal for GENERIC_PLAN.md. Angle: **micro-frontend / module-federation** — each
feature module is an independently built, independently deployed federated remote; the host app is
a Module Federation host that resolves, verifies, and loads remote containers at runtime.

## 1. Summary

The app is split into a **host shell** (navigation, auth, design system, all native code, the
module-SDK bridge) and **federated remote modules**, each built as a standalone Module Federation
v2 container by **Re.Pack 5** (Rspack-based bundler for React Native, by Callstack). The host ships
through the app stores; remotes are plain-JS MF containers published to a module registry and
downloaded at runtime by Re.Pack's `ScriptManager` with hash + signature verification before
evaluation. Shared dependencies (`react`, `react-native`, `expo`, every `expo-*` package the host
embeds) are declared as **eager singletons provided by the host**; remotes consume them from the MF
share scope and never bundle their own copies. AI agents develop modules in standalone sandbox
repos generated from a template, running against a dev build of the *real* host shell that loads
the module from `localhost` — the exact same loading path as production, just a different resolver.

**What makes this approach distinct:** modules are *real independently-built artifacts* with an
industry-standard container format (Module Federation v2), not slices of one monorepo bundle (the
Metro-bundle-splitting sibling) and not a custom interpreter/DSL (the other sibling). Build
isolation is total — a module's build cannot break the host build, and the host never rebuilds when
a module ships. The price: we leave Metro for the bundler-critical path, which is the roughest edge
of Expo integration, called out honestly in §3 and §10.

## 2. Architecture Overview

```
┌──────────────────────────── HOST APP (app-store binary) ────────────────────────────┐
│  Rspack/Re.Pack host build (Hermes HBC main bundle)                                  │
│                                                                                      │
│  ┌─────────────┐  ┌──────────────────┐  ┌──────────────────────────────────────┐    │
│  │ Shell UI     │  │ Module SDK bridge│  │ MF v2 runtime + Re.Pack ScriptManager │    │
│  │ React Nav,   │  │ @acme/module-sdk │  │  resolver → registry URL              │    │
│  │ auth, theme  │  │ (capability-     │  │  cache (FS) → verify sha512+ed25519   │    │
│  └─────────────┘  │  scoped APIs)    │  │  → evaluate container → share scope   │    │
│  ┌─────────────┐  └──────────────────┘  └──────────────────────────────────────┘    │
│  │ Native: all expo-* / react-native modules, embedded ahead of time               │ │
│  └──────────────────────────────────────────────────────────────────────────────── ┘ │
└───────────────────────────────▲──────────────────────────────────────────────────────┘
                                │ HTTPS: GET /v1/apps/:app/manifest?channel=prod&host=42
                                │        GET /cdn/modules/shop/1.4.2/shop.container.js.bundle
┌───────────────────────────────┴───────────────┐      ┌───────────────────────────────┐
│  MODULE REGISTRY (API + Postgres)             │      │  SANDBOX (per AI agent)        │
│  channels: dev / staging / prod               │      │  module repo from template     │
│  targeting, staged rollout, kill-switch       │◄─────│  rspack serve :9000 (remote)   │
│  artifacts → S3/R2 + CDN, immutable, signed   │ pub  │  host-shell dev build loads it │
└───────────────────────────────────────────────┘      └───────────────────────────────┘
```

- **Host app**: built with Re.Pack (`@callstack/repack` 5.x) instead of Metro. Contains
  `Repack.plugins.ModuleFederationPluginV2` configured as host, the full native dependency set,
  and the SDK bridge. Baseline modules can be bundled into the binary as "local remotes" (container
  files in app assets) for first-run/offline.
- **Module packaging**: each module is its own repo/workspace, built by Re.Pack as a remote — output
  is `<name>.container.js.bundle` + exposed chunks + `mf-manifest.json` + assets, zipped with our
  module manifest into an `.fmod` artifact (a tar.zst).
- **Delivery**: registry API resolves "which module versions for this user/channel/host-build",
  returns CDN URLs; `ScriptManager` downloads, caches on the filesystem, verifies, evaluates.

## 3. Module Format & Loading Mechanism

**Bundle format.** A module artifact (`.fmod`) contains:

```
shop-1.4.2.fmod
├── module.json              # our manifest (below)
├── mf-manifest.json         # standard MF v2 manifest (exposes, shared, remoteEntry)
├── shop.container.js.bundle # MF container entry (plain JS, evaluated by Hermes)
├── chunks/*.chunk.bundle    # async chunks within the module
├── assets/...               # images, fonts (no native code permitted)
└── SIGNATURE                # ed25519 over module.json, which carries per-file sha512
```

**Manifest schema sketch (`module.json`):**

```jsonc
{
  "id": "shop",
  "version": "1.4.2",
  "entry": "shop.container.js.bundle",
  "exposes": { "./Screen": "Screen", "./routes": "routes" },
  "hostApi": "^2.1.0",                       // SDK bridge semver contract (§8)
  "hostProfile": "host-42",                   // shared-dep snapshot built against (§8)
  "capabilities": ["storage.scoped", "network.fetch:api.acme.com", "nav.push"],
  "sharedConsumed": { "react": "19.1.0", "react-native": "0.81.x", "expo": "~54.0.0" },
  "files": { "shop.container.js.bundle": { "sha512": "..." }, "chunks/...": { "sha512": "..." } },
  "minHostBuild": 42, "killSwitchable": true
}
```

**Runtime injection.** Host startup registers a resolver:

```ts
ScriptManager.shared.addResolver(async (scriptId, caller) => {
  const m = registryClient.resolved(scriptId);            // from /manifest response
  return { url: Script.getRemoteURL(m.cdnUrl), cache: true, verifyScriptSignature: 'strict' };
});
```

Mounting a module: `const Screen = React.lazy(() => loadRemote('shop/Screen'))` (from
`@module-federation/runtime`, which Re.Pack 5 embeds), rendered inside
`<ModuleBoundary id="shop">` — an error boundary + render-watchdog that reports to the registry
and flips the local kill-switch on repeated crash, satisfying the "never takes down the host"
requirement. Unload = unmount + evict container from MF runtime cache; "update" = unload, evict FS
cache, re-resolve. Container evaluation happens via Re.Pack's script evaluation on the native side
(`ScriptManager` hands the file to React Native's JS bundle loader), with our verification hook
(sha512 of every file + ed25519 signature against a pinned public key) running **before** eval.

**Shared-dependency dedup.** Host config: `shared: { react: { singleton: true, eager: true,
version: '19.1.0' }, 'react-native': {...}, expo: {...}, /* every expo-* generated from host
package.json by a script */ }`. Module config: same keys with `import: false` (consume-only) and
`requiredVersion` ranges — so remote bundles contain *zero* copies of React/RN/Expo; the MF v2
runtime resolves them from the host's share scope at load time and rejects the load (graceful, not
crash) if `requiredVersion` is unsatisfiable. This is the standard MF singleton contract and is the
mechanism that keeps module bundles small (~tens of KB) and prevents the classic "two Reacts"
hooks crash.

**Honest rough edges (Expo is Metro-centric):**

- `npx expo start` / `npx expo run` drive Metro only. With Re.Pack we use Re.Pack's dev server and
  `react-native run-ios/android` (or direct Xcode/Gradle); `expo prebuild`, config plugins, and EAS
  Build still work because they are bundler-agnostic on the native side, but we must set
  `bundleCommand`/Gradle hooks to Re.Pack's bundler. Dev-client UX (expo dev menu integration) is
  partially lost.
- **expo-router is effectively off the table**: it depends on Metro's `require.context` over the
  `app/` directory and Metro-specific resolution (confirmed in `packages/expo-router/src`, which
  ships a require-context ponyfill only for tests). We use React Navigation directly; modules
  export route descriptors through the SDK instead.
- `expo-updates` assumes Metro-produced bundles and its own manifest protocol; it can still update
  the *host* base bundle, but mixing it with federated remotes is confusing — simpler to let our
  registry own all OTA and drop expo-updates.
- Remote containers are **plain JS, interpreted by Hermes** (no ahead-of-time HBC for remotes —
  Hermes cannot `eval` bytecode; loading `.hbc` chunks needs a custom native JSBundleLoader path,
  deferred to Phase 3). Expect measurable parse/eval cost on large modules; budget enforced in CI.
- Some `expo-*` packages assume Metro for asset resolution; Re.Pack has an assets loader, but each
  SDK package the host exposes must be smoke-tested under Rspack once.

## 4. Sandbox Strategy for AI-Agent Development

- `acme create-module shop` scaffolds a standalone repo: TS + the module's `rspack.config.mjs`
  (remote preset, shared map auto-generated from the target **host profile**, §8), `module.json`,
  Jest, ESLint with our rule pack, and `@acme/module-sdk` as a dependency — **types and a Jest mock
  implementation only**; the real implementation lives in the host and arrives via the share scope.
- The agent runs `acme dev`: starts Re.Pack dev server on :9000 serving the remote, and installs a
  prebuilt **host-shell dev binary** (the real host app, dev flavor) on the simulator whose
  resolver is pointed at `http://localhost:9000` for this module id. The agent iterates with HMR,
  drives the simulator (argent/Maestro), and runs unit tests against the SDK mock.
- **Failure containment is structural**: the module repo has no filesystem or git access to the
  host repo; the host binary is a fixed artifact the agent cannot rebuild; the only contact surface
  is the typed SDK plus capability grants declared in `module.json` and enforced by the host bridge
  at runtime. Worst case, the agent ships a module that crashes its own `ModuleBoundary` in a dev
  channel.
- **Sandbox ≡ production**: the dev loop uses the identical `ScriptManager` path, share scope, and
  manifest schema as production — the only delta is resolver URL and signature mode
  (`verifyScriptSignature: 'off'` in dev). What loads in the sandbox loads in prod by construction.

## 5. Backend / Registry Design

- **Storage**: Postgres for metadata (`modules`, `versions`, `channels`, `rollouts`,
  `host_profiles`, `audit_log`); artifacts in S3/R2 behind a CDN, immutable, content-addressed
  (`/modules/<id>/<version>/<sha>/...`).
- **API** (small Fastify/Workers service):
  - `GET /v1/apps/:app/manifest?channel&hostBuild&userId` → list of resolved module versions +
    CDN URLs + signatures; the resolver filters on `hostApi`/`hostProfile` compatibility and
    targeting rules (per-user allowlist, cohort hash, percentage rollout).
  - `POST /v1/modules/:id/versions` (CI only, OIDC-authenticated) → upload + server-side
    signature with an ed25519 key in KMS; the matching public key is baked into the host binary.
  - `POST /v1/channels/:channel/promote`, `POST /v1/rollback` (repoint channel to a previous
    immutable version — instant, no rebuild), `POST /v1/killswitch/:moduleId`.
- **Channels**: `dev` (sandbox builds, auto-published per agent branch), `staging`, `prod`.
  Side-by-side versions per channel; dev channel entries can point at ephemeral artifacts.
- Client telemetry endpoint ingests per-module crash/error counts to drive health-gated rollouts.

## 6. Verification & Promotion Pipeline

1. **CI (module repo, every push)**: `tsc --noEmit`, ESLint (incl. custom rules: no
   `react-native` deep imports, no `NativeModules`/`TurboModuleRegistry` references, SDK-only
   imports), Jest against SDK mock, Re.Pack production build, **post-build bundle scan** (regex/AST
   over the emitted container for forbidden globals — belt-and-suspenders since the lint runs
   pre-bundle), bundle-size budget, shared-map check (zero duplicated shared deps in output).
2. **Runtime smoke test**: CI boots the host-shell binary in a simulator, loads the freshly built
   container through the real `ScriptManager`, runs a Maestro flow exported by the module
   (`flows/smoke.yaml`), asserts no `ModuleBoundary` trip and a screenshot diff baseline.
3. **AI review agents**: (a) code reviewer diffs against the SDK contract and module guidelines;
   (b) behavioral reviewer replays the Maestro flow and judges screenshots against the feature
   spec; both post structured verdicts to the registry as required checks.
4. **Human gate**: owner approves in the registry dashboard → artifact is signed and promoted
   `dev → staging`.
5. **Staged rollout to prod**: 1% → 10% → 50% → 100% with automatic halt + rollback if the
   module's crash rate or boundary-trip rate exceeds threshold; manual instant rollback and global
   kill-switch always available (host checks kill-switch flags on manifest refresh and on launch).

## 7. Security & App-Store Compliance

- **JS-only OTA**: remotes are interpreted JavaScript executed by the embedded Hermes runtime; no
  native code ever ships OTA — same legal footing as EAS Update/CodePush under Apple's
  Developer Agreement §3.3.1(B) (interpreted code that doesn't change the app's primary purpose)
  and Google Play's device-and-network-abuse policy. Modules extend features of the reviewed app;
  we enforce an internal policy review that no module changes the app's primary purpose.
- **Integrity**: per-file sha512 in `module.json`, ed25519 signature over the manifest, verified
  before evaluation; TLS for transport; signing key in KMS, public key pinned in the binary,
  with a key-rotation slot (two pinned keys).
- **Capability scoping**: modules never touch `NativeModules`; the SDK bridge proxies storage
  (namespaced per module), network (host-enforced domain allowlist from `capabilities`),
  navigation, and native features, checking the manifest's grants at call time.
- **Honest limit**: all modules share one Hermes runtime, so JS-level isolation is *advisory* —
  a malicious module that evades the bundle scan can reach host globals. Mitigations: we control
  authorship (our agents + CI + signing, not third-party code), `Object.freeze` on bridge
  surfaces, and a Phase-3 option to move untrusted modules into a separate runtime
  (`react-native-webview` JS context or a second Hermes instance) at IPC cost.

## 8. Versioning / Compatibility Contract

- **`hostApi` (semver)**: the SDK bridge surface. Host advertises e.g. `2.3.0`; module declares
  `"hostApi": "^2.1.0"`. Registry filters server-side; host re-checks client-side and refuses to
  evaluate incompatible modules (shows the module's "unavailable" fallback instead).
- **Host profile = shared-dependency snapshot**: every host release publishes
  `host-profile.json` (exact versions of all shared singletons: react 19.1.0, react-native 0.81.x,
  expo ~54, each expo-*). Module CI builds **against a named profile**, recorded in the manifest.
  The registry only serves a module to host builds whose profile satisfies the module's
  `sharedConsumed` ranges — version skew is resolved server-side first.
- **Second line of defense**: MF v2 runtime enforces `requiredVersion` per shared package at load;
  an unsatisfied singleton makes `loadRemote` reject (caught by `ModuleBoundary`), never a
  silent dual-React.
- **Skew policy**: host upgrades that bump a shared major create a new profile; old modules keep
  being served to old host builds (artifacts are immutable), and CI opens automated "rebuild
  against profile N+1" PRs in each module repo. Support window: latest two host profiles.

## 9. MVP Roadmap (3 Phases)

- **Phase 1 — Federated host walking skeleton (~5–6 weeks, 2 eng)**: convert an Expo prebuild app
  to Re.Pack 5 (host MF config, React Navigation, EAS Build integration via custom bundle
  command); one hand-written remote module loaded from a static URL with `ScriptManager`;
  shared-singleton map generated from package.json; `ModuleBoundary`; smoke-test the embedded
  expo-* set under Rspack. Exit: module updates without an app build.
- **Phase 2 — Registry + sandbox (~6–8 weeks, 2–3 eng)**: registry API/DB/CDN, channels,
  signing + client verification, `acme create-module` / `acme dev` CLI, host-shell dev binary
  distribution, module-repo CI with bundle scan and simulator smoke test, manual promote/rollback
  dashboard. Exit: an AI agent ships a module from empty repo to `staging` without human keyboard.
- **Phase 3 — Production hardening (~6–8 weeks, 2–3 eng)**: AI review agents as required checks,
  staged rollout with health gates and auto-rollback, per-module observability (Sentry tags,
  metrics), offline cache + embedded baseline modules, host-profile skew automation, evaluate HBC
  loading for remotes via custom native JSBundleLoader.

## 10. Key Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Expo/Metro divergence: no `expo start`, no expo-router, expo-updates dropped; some expo-* untested under Rspack | High — ongoing friction with every Expo SDK upgrade | Pin Expo SDK per host profile; Phase-1 compatibility test matrix for the embedded expo-* set; React Navigation from day one; keep prebuild/config-plugins (bundler-agnostic) as the native pipeline |
| Re.Pack/MF-runtime dependency on Callstack's maintenance & RN release lag | Medium — blocked upgrades | Re.Pack is actively maintained and Callstack-commercial; isolate via thin wrapper around `ScriptManager`/`loadRemote` so a swap to a custom loader stays feasible |
| Plain-JS eval of remotes on Hermes (no HBC) hurts module TTI | Medium | CI bundle-size budgets; lazy `loadRemote` per screen; chunk splitting inside modules; Phase-3 HBC loader |
| Same-runtime "sandbox" is soft; a hostile module can escape capability scoping | Medium (low while all authors are internal agents + CI + signing) | Bundle scanning, frozen bridge, signed artifacts, kill-switch; separate-runtime option in Phase 3 if third-party modules ever land |
| Version-skew matrix (modules × host profiles) grows | Medium | Server-side profile filtering, two-profile support window, automated rebuild PRs |
| App-store policy drift on downloadable code | High but industry-wide | JS-only invariant enforced in CI; modules never change primary purpose; same exposure as CodePush/EAS Update |
