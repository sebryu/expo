# RSC_ANALYSIS — Can React Server Components Replace the Module-Delivery Mechanism in REVIEWED_SPEC?

Analysis date: June 2026. Grounded in this monorepo (expo `56.0.5`, expo-router `56.2.7`, expo-server `56.0.4`) — sources cited inline.

---

## Verdict

**Not suitable as a full replacement. Conditionally suitable as a hybrid for a narrow subset of module types — and even that is premature today.**

RSC fails the spec's core requirement (G1): it **cannot deliver new client-side interactive code** after the binary ships. The client-component module map is frozen at `expo export` time, so an updated server can only recompose interactivity that already exists in the shipped client bundle. RSC also fails offline (H9) for request-time rendering, inverts the spec's version-skew model (C7/H3) by requiring lockstep binary↔server-deployment pairs, and — per Expo's own docs and source in this repo — is still experimental, explicitly "not recommended" for production, and incompatible with EAS Update. The spec's planned "download JS bundle, verify, evaluate in Hermes" loader remains necessary; notably, Expo's *own* RSC machinery uses exactly that mechanism (fetch + `eval` of split chunks) internally, validating the spec's approach.

---

## The crux: the `"use client"` boundary (requirement 1)

How Expo RSC resolves client references, verified in source:

- The RSC payload (react-server-dom-webpack flight format) references client components by **module ID minted at build time**.
- On device, IDs resolve via `packages/@expo/metro-runtime/rsc/runtime.js`:
  - `__webpack_require__ = (id) => __r(id)` — Metro require **from the already-shipped bundle**;
  - `__webpack_chunk_load__ = (id) => __loadBundleAsync(id)` — fetch + `eval` of a **split chunk produced at export time**.
- At production export, every discovered `"use client"` boundary is injected into the client bundle through the virtual module `expo/virtual/rsc.js` as `{ [require.resolveWeak(b)]: () => import(b) }` (`packages/@expo/metro-config/src/transform-worker/transform-worker.ts:74-103`). The CLI walks server bundles to collect `reactClientReferences` (`packages/@expo/cli/src/start/server/metro/createServerComponentsMiddleware.ts`).

**Consequence:** the set of client components — i.e., all stateful, gesture-handling, hook-using, native-API-touching code — is **capped by what the binary build saw**. A new server deployment can:
- recompose / re-prop / re-style existing client components (yes);
- ship arbitrary new *server* component trees: static markup, data, layout (yes);
- attach interactions as Server Functions — but each press is a network POST round trip (`callServerRSC` in `packages/expo-router/src/rsc/router/host.tsx`), unusable for real interactivity (drag, local state, animation) and dead offline;
- introduce a genuinely new `"use client"` component (**no** — no module ID, no chunk, no map entry).

The runtime even hardens for this exact failure: `runtime.js` disables RN fatal-error handling in production so a *missing client module* throws into the nearest error boundary instead of crashing — i.e., "server references unknown client code" is an anticipated error state, not a capability.

A spec module is by definition a new interactive feature (screens with local state, gestures, SDK calls). **RSC cannot deliver that; gap is fundamental, not a maturity issue.**

---

## Capability-gap table

| Spec requirement | RSC capability (Expo, June 2026) | Gap? |
|---|---|---|
| G1: new interactive features w/o store release | Only recomposition of build-time client components; new interactivity requires new client bundle (EAS Update — itself incompatible with RSC today) | **Fundamental gap** |
| H9: cached modules work offline | Native RSC fetches use explicit no-cache headers (`host.tsx` `NO_CACHE_HEADERS`); no payload persistence (TODO in source: "Load from on-disk on native when indicated"); only `render: 'static'` build-time payloads embed in the binary, and full-RSC static output "doesn't fully work yet" (docs) | **Gap** |
| H5/H6: containment + kill-switch | Comparable containment: server errors → `ReactServerError` → error boundary; kill = stop serving the subtree (instant, no manifest needed). Actually slightly *better* for kill | No gap |
| B3/P11: per-module instant rollback | Unit of rollback is the whole server deployment, not a module; needs per-feature server routing discipline to approximate | Partial gap |
| §6: AI sandbox velocity | Server components unit-test in plain Node (`jest-expo/rsc/{ios,android,web}` presets, `docs/pages/guides/testing-rsc.mdx`) — genuinely easier for markup-level work; but Tier-2 fidelity still needs simulator, and agents are barred from writing any new interactive code at all | Mixed |
| C7/H3: semver skew (many module versions × hosts) | Inverse model: payload format couples server to exact client bundle; Expo's answer is 1:1 **versioned server deployments pinned per binary** (`EXPO_UNSTABLE_DEPLOY_SERVER=1`, alpha — `docs/pages/router/web/api-routes.mdx`) | **Gap** (model mismatch) |
| K1: store compliance | RSC payload is data ("custom JSON-like format"), cleaner than downloaded code; split-chunk fetch+eval sits on the same legal footing as EAS Update | No gap (slightly better) |
| H10: lazy, low-latency UX | Every dynamic screen entry = network round trip + Suspense fallback (no-cache on native prod); streaming is nice for content; local cached bundle wins after first download | Gap for app-like UX |
| Production readiness | `experiments.reactServerFunctions` / `reactServerComponentRoutes` flags (`@expo/config-types/src/ExpoConfig.ts:274-279`); docs: beta/experimental, "Production deployment is limited and not recommended yet" | **Gap today** |

---

## Other requirements in detail

**Offline (2).** `fetchRSC` → `expo/fetch` with `Cache-Control: no-cache` on native production; failures surface as `NetworkError`. There is no payload cache layer to persist or replay. The only offline path is build-time (`render: 'static'`) payloads embedded in the binary — which, by being build-time, cannot deliver post-release features either. Spec H9 fails for anything RSC delivers dynamically.

**Containment & kill (3).** Roughly equivalent to the spec's error-boundary plan; server-driven kill is *simpler* (no manifest refresh cycle — next fetch just omits the feature). But the circuit-breaker/local-disable semantics (H5) would still be client-side work either way.

**AI sandbox (4).** Real upside: agents writing *server* components iterate in Node with jest-expo RSC presets, no device. Real downsides: a second execution environment (`react-server`) with sharp edges agents will hit constantly — on native server components `StyleSheet.create` and `Platform.OS` don't work (use plain style objects, `process.env.EXPO_OS`), nested Server Function calls break on Hermes, libraries need `"use client"` interop shims. Net: easier for content modules, harder overall.

**Version skew (5).** The spec wants N module versions × M host versions governed by a semver SDK contract. RSC's flight format + build-minted module IDs force exact pairing; Expo pins each binary to its own immutable server deploy. You would run one live server deployment per binary version in the wild, and "rollback module X" becomes "redeploy the paired server" — coarser than B3.

**Performance & UX (7).** Streaming Suspense is excellent for content feeds; for app-like modules, request-time rendering on every navigation (enforced no-cache on native) is strictly worse than a locally cached, lazily evaluated bundle (H10's <50 ms loader-overhead target is unreachable for network-bound rendering).

---

## What a hybrid would look like (if pursued)

Keep the spec's loader as the **primary** mechanism; treat server-driven UI as an optional module *type*:

1. **Downloaded-bundle modules (primary, as specced):** all interactive features. Note convergence: the loader can reuse the exact pattern Expo ships — `loadBundleAsync` → fetch → `eval` in Hermes (`packages/expo/src/async-require/`), plus the spec's hash/signature layer on top.
2. **Server-rendered content modules (optional subset):** content-heavy, inherently online surfaces (feeds, dashboards, promos, settings-driven content) rendered server-side against a **host-shipped client-component kit** (the design system = the `"use client"` palette). Kill/rollback per feature falls out of server routing. Manifest entry type: `server-ui` with a URL instead of an artifact.
3. **Pragmatic v1 alternative for (2):** a small bespoke JSON→design-system SDUI renderer delivered *as a normal module* gives the same server-driven benefits without adopting experimental RSC infra, dual runtimes, or the versioned-deployment coupling. Recommended over RSC until it stabilizes.
4. If RSC is adopted later for (2), its constraint set is acceptable there precisely because those modules are online-only and non-interactive beyond the shipped kit — but it must never become the path for interactive modules.

---

## Maturity assessment (this repo, June 2026)

- Both flags remain under `experiments` in `@expo/config-types` ("Experimentally enable…"); the guide (`docs/pages/guides/server-components.mdx`) is marked experimental/beta, SDK 52+ — still not on by default four SDK releases later (SDK 56).
- Known-limitations list (same doc, current): **"EAS Update does not work with Server Components yet"**, **"Production deployment is limited and not recommended yet"**, RSC→HTML SSR unsupported (static/server web output incomplete), DOM components can't use Server Functions in production, form integration incomplete, Hermes nested-Server-Function limitation, `StyleSheet.create`/`Platform.OS` unsupported on native server.
- Full RSC routes mode (`reactServerComponentRoutes`): "Stack, Tabs, and Drawer do not support Server Components yet", most `Link` props unsupported — the router itself is mid-rewrite for it.
- Native server deployment/pinning is **alpha** (`EXPO_UNSTABLE_DEPLOY_SERVER`, api-routes.mdx).
- Active churn through 2025–2026: RSC logic migrated out of expo-router into `@expo/router-server` (#40484), component-ID minting canonicalized (#45900), actions moved to POST-only (#45905), render-context internals removed (#45908) — internals are still being reshaped between minor releases.

Building the spec's day-one kill/rollback/store-shipping requirements on this foundation would stack an experimental dependency under a system whose whole premise is containment and operational safety.

---

## Bottom line

RSC answers a different question ("how do I run React on the server and stream UI to a known client?") than the spec asks ("how do I ship new, verified, independently revocable interactive code to an existing client?"). Use the planned bundle loader; revisit RSC only as a later, optional module type for online content surfaces once it leaves experimental status.
