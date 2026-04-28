# Phase 0 Research: VARS — MVP

Research findings for the technical-context decisions recorded in `plan.md`. Each section follows: **Decision / Rationale / Alternatives**.

All `NEEDS CLARIFICATION` items raised during plan drafting are resolved here.

---

## R1 — UI framework (or none)

**Decision**: **Vanilla TypeScript** at MVP. No React, Preact, Svelte, or Vue. Revisit only if state/render complexity exceeds what hand-written DOM + Canvas can carry.

**Rationale**:
- The MVP UI is dominated by Canvas (schematic + PCB views) plus a handful of HTML overlays (multimeter readout, inventory drawer, status). Frameworks add minimal value when the dominant render surface is Canvas, since they don't help you draw on it.
- Bundle budget per Constitution Principle II is **<250 KB gzipped**. Preact (~4 KB) and Svelte (~3 KB compiled) aren't blockers, but every kilobyte helps cold-start on mobile, and at MVP scope we don't need framework state management.
- Vanilla also keeps the dependency surface small and the build fast.

**Alternatives considered**:
- **Preact**: ~4 KB gzipped. Reasonable fallback if HTML-overlay state grows post-MVP. Easy to retrofit since we're staying pure.
- **Svelte**: Smallest compiled bundle and excellent DX, but the build complexity (compiler + HMR boundary) outweighs the win for MVP scope.
- **React**: Rejected — bundle size + runtime cost out of line with Principle II budgets.

---

## R2 — MNA solver: roll our own vs. JS library

**Decision**: **Roll our own** custom dense MNA solver in `src/solver/`. Sparse representation deferred — MVP circuits have ≤10 nodes, and even a 20-node circuit's dense matrix is 400 doubles, which LU-decomposes in microseconds.

**Rationale**:
- No JS library exists with a clean MNA-specific API. The closest options (`mathjs`, `numeric.js`) are either too heavy or unmaintained.
- The solver is the project's intellectual core (Constitution Principle I, NON-NEGOTIABLE). Owning it means we control accuracy, can validate against analytical answers in unit tests, and can profile/optimize for mobile.
- A clean MNA implementation is roughly 200–400 lines of TypeScript: a build step (component-by-component conductance stamping), a solve step (LU decomposition with partial pivoting), and a result-extraction step.

**Alternatives considered**:
- **`mathjs`**: Rejected — adds ~150 KB minified. Solving 5×5 to 20×20 matrices doesn't justify pulling in a full math toolkit.
- **`numeric.js`**: Rejected — unmaintained since ~2014; security & compat risk.
- **Embedded SPICE engine** (e.g., a WASM-compiled ngspice): Rejected — Constitution Principle IV prohibits SPICE-like surfaces for users; using one as a hidden engine pulls in WASM blobs that hurt mobile cold-start and adds operational risk.
- **`gl-matrix` or similar**: General matrix lib, not solver-shaped. Rejected — same reason as `mathjs`, less fit.

---

## R3 — Web Audio API path (post-MVP)

**Decision**: **AudioWorklet** with a single processor that consumes simulation state via a SharedArrayBuffer (or via main-thread `port.postMessage` if SAB is unavailable on the target browser) and synthesizes the output. **Not** ScriptProcessorNode — deprecated. Detailed design lives in the spec for the audio-capable circuit's release.

**Rationale**:
- `ScriptProcessorNode` is deprecated; runs on the main thread; jitter under JS GC pauses is incompatible with Principle V's "feedback reflects solver state" promise.
- AudioWorklet runs on the dedicated audio thread, giving consistent low-latency playback.
- SharedArrayBuffer between main and worklet keeps the worklet decoupled from main-thread GC. Where SAB is unavailable (e.g., older Safari without COOP/COEP headers — irrelevant for a same-origin PWA shell, but still), fall back to `port.postMessage` snapshots at a coarser update rate.

**Alternatives considered**:
- **ScriptProcessorNode**: Rejected — deprecated, main-thread only.
- **MediaElementSourceNode + pre-rendered audio**: Rejected — Constitution Principle I forbids pre-recorded audio for nominal operation; output must reflect solver state.
- **OfflineAudioContext**: Useful for tests (deterministic offline rendering) but not for live playback.

---

## R4 — Schematic + PCB rendering: Canvas vs SVG

**Decision**: **HTML5 Canvas** for both views. Annotations (test-point labels, expected voltages, component IDs) are rendered as Canvas text, not DOM, so zoom/pan stays trivially aligned with the schematic.

**Rationale**:
- Canvas is faster than SVG on mobile when many small visual elements need redrawing each frame (probes, hover halos, dust/corrosion in later releases).
- SVG would be ergonomically nicer for static schematics, but its hit-testing and DOM weight scale poorly on phones once components number in the dozens.
- Touch interaction (pointer events) attaches naturally to a single Canvas element with custom hit-testing against the circuit's data-model coordinates.

**Alternatives considered**:
- **SVG**: Rejected for MVP. May reconsider for accessibility (DOM-based screen-reader support is far better with SVG) once a11y becomes a v1.x priority.
- **WebGL**: Rejected — Constitution mandates 2D / 2.5D only; overkill for the MVP visual complexity and adds GPU-driver compatibility risk on older mobile.
- **Hybrid (SVG schematic + Canvas overlay)**: Reasonable post-MVP option for accessibility; defer.

---

## R5 — PWA setup: plugin vs hand-rolled service worker

**Decision**: **`vite-plugin-pwa`** with `generateSW` strategy and `precacheAndRoute` of all assets emitted by Vite. Manifest in `public/manifest.webmanifest`. Workbox runtime caching uses `CacheFirst` for fonts/icons and `StaleWhileRevalidate` for the manifest itself.

**Rationale**:
- The plugin is battle-tested, integrates cleanly with Vite's asset hashing, and gives offline precaching for free.
- Workbox handles the cache-invalidation minefield that plagues hand-rolled service workers.
- Constitution Principle III demands the app be functional offline after first load. Precaching all build outputs achieves this in a single line of config.

**Alternatives considered**:
- **Hand-rolled service worker**: Rejected — cache invalidation and update flow are notoriously bug-prone; the plugin saves weeks.
- **No service worker, only IndexedDB**: Rejected — IndexedDB persists *data*, not the *app shell*. Without a service worker, the second visit fails offline.

---

## R6 — Touch interaction model

**Decision**: **Vanilla Pointer Events**. No external gesture library. MVP interaction vocabulary:

- **Tap** (pointerdown + pointerup, <300 ms, <10 px movement) — place probe / select component.
- **Long-press** (pointerdown held >500 ms, <10 px movement) — remove a component from the board.
- **Drag** — placing a replacement from inventory onto an empty board position; lands when drag-replace gameplay is implemented (Story 2 acceptance).

**Rationale**:
- Pointer Events are universally supported on the target platforms.
- Touch libraries (Hammer.js, ZingTouch) add weight for a gesture vocabulary the MVP doesn't need.
- Custom hit-testing against the circuit data model is a few dozen lines.

**Alternatives considered**:
- **Hammer.js**: Rejected — bundle weight (~12 KB) for two gestures we can implement in 50 lines.
- **PointerEvent polyfill**: Not needed on target browsers.

---

## R7 — Mobile bundle-budget enforcement

**Decision**: Add a CI check that fails if the gzipped JS bundle exceeds **250 KB**. Tooling: a small custom Node script in `scripts/check-bundle-size.mjs` that reads Vite's build manifest and `gzip`s each chunk. Optionally `bundlesize` if it's lighter than rolling our own.

**Rationale**:
- Without an enforcement mechanism, bundles drift up over time. Constitution Principle II is operational only if the budget is checked in CI.
- The custom script avoids another npm dependency for a 30-line task.

**Alternatives considered**:
- **`bundlesize`**: Reasonable, but unmaintained as of late 2024. Roll our own.
- **`size-limit`**: Maintained, ~20 transitive deps. Acceptable fallback if rolling our own gets clunky.

---

## R8 — >50-node solver performance (open question from spec)

**Decision**: **Defer.** MVP circuit is ≤10 nodes; well within budget. Establish a CI perf test fixture in `tests/unit/solver/perf.spec.ts` that runs a 50-node synthetic circuit and asserts `<50 ms p95` solve time. If post-MVP circuits hit this ceiling, migrate to a sparse solver (CSR-format) running in a Web Worker.

**Rationale**:
- Premature optimization for a problem the MVP doesn't have.
- The CI budget catches the regression before users do.

**Alternatives considered**:
- **Pre-emptive sparse implementation**: Rejected — adds complexity without an MVP-level need.
- **Web Worker from day one**: Rejected — message-passing overhead dominates at <50-node sizes; main-thread solve is faster for MVP scope.

---

## R9 — Persistence schema versioning

**Decision**: IndexedDB stores carry a top-level `schemaVersion: number` on each record. Migrations run on `IDBOpenDBRequest.onupgradeneeded` based on `oldVersion → newVersion`. v1 ships with `schemaVersion: 1`.

**Rationale**:
- Constitution Principle III demands offline durability, which means real users will have stored data when v1.x ships changes. A migration plan from day one is cheap insurance.
- IndexedDB's native `versionchange` mechanism handles store creation; we add a per-record `schemaVersion` so individual record migrations can run lazily on read where helpful.

**Alternatives considered**:
- **No version field; rely only on IndexedDB version**: Rejected — coarse-grained; can't migrate one store at a time.

---

## R10 — Type-safety boundary for circuit JSON

**Decision**: Use **`zod`** (or hand-written validators if the bundle cost is meaningful) to validate every loaded `CircuitTemplate` against the JSON-Schema in `contracts/circuit-template.schema.json`. Fail loudly on validation errors at boot; never load an invalid circuit.

**Rationale**:
- Curated templates are data; nothing in the runtime catches a schema typo unless we validate.
- `zod` is ~12 KB gzipped. Acceptable tradeoff for runtime safety.

**Alternatives considered**:
- **Trust TypeScript types alone**: Rejected — types disappear at runtime; a typo in `circuits/001-voltage-divider.json` would surface as a confusing solver error rather than a clear validation error.
- **Hand-rolled validators**: Acceptable if `zod`'s 12 KB becomes a problem; defer optimization.

---

**All NEEDS CLARIFICATION resolved.** Plan ready for Phase 1 design.
