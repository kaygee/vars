# Implementation Plan: Vintage Amp Repair Sim (VARS) — MVP

**Branch**: `master` (path-B in-place; no dedicated feature branch) | **Date**: 2026-04-28 | **Spec**: [./spec.md](./spec.md)
**Input**: Feature specification from `specs/001-vintage-amp-repair-sim/spec.md`

## Summary

VARS is delivered as a **mobile-first Progressive Web App** built with TypeScript, Vite, and a hand-rolled Modified Nodal Analysis (MNA) solver. The MVP ships a single voltage-divider-tier circuit that exercises **User Story 1** (probe + diagnose) and **User Story 2** (replace + verify) end-to-end. **User Story 3** (audio + contact-cleaner) defers to the first post-MVP release that ships a circuit with an audio output stage. Persistence is local-only via IndexedDB; the entire app must work offline after first load (Constitution Principle III).

Core technical bets:

- **Custom TypeScript MNA solver** in `src/solver/` — owning Principle I (physical accuracy non-negotiable). Dense matrices at MVP scale (≤10 nodes); sparse + Web Worker migration deferred until needed.
- **HTML5 Canvas** for schematic + PCB views — fastest path to 60 fps on mobile and the cleanest target for custom hit-testing.
- **Vanilla TypeScript** at MVP — no React/Preact/Svelte. The dominant render surface is Canvas; framework cost (bundle + runtime) doesn't pay for itself yet.
- **Web Audio AudioWorklet** (deferred to post-MVP audio circuit) — `ScriptProcessorNode` is deprecated.
- **Vite 5 + `vite-plugin-pwa`** (Workbox) for the offline shell. Manifest in `public/`.
- **IndexedDB via `idb`** for repair sessions, parts inventory, and any cross-session state.
- **Vitest** for solver and pure-logic units; **Playwright** for touch-driven and PWA-install / offline browser tests.

## Technical Context

**Language/Version**: TypeScript 5.4+ targeting ES2022 modules
**Primary Dependencies**: Vite 5.x (build/dev), Vitest 1.x (unit), Playwright 1.x (e2e), `vite-plugin-pwa` (Workbox-backed service worker), `idb` (small IndexedDB wrapper). No UI framework at MVP.
**Storage**: IndexedDB via `idb`. No LocalStorage (size + sync semantics worse). No backend.
**Testing**: Vitest for `src/solver/`, `src/components/`, `src/session/` (all pure-logic / DOM-free). Playwright for `tests/e2e/` (touch input, PWA install, airplane-mode behavior).
**Target Platform**: Modern mobile browsers — Safari iOS 16+, Chrome Android last-2-major, Firefox Android last-2-major. PWA-installable on Android (Chrome) and iOS (Add-to-Home-Screen). Desktop browsers work but are not the optimization target.
**Project Type**: Single web project (frontend-only, no backend, no separate package).
**Performance Goals**: 60 fps steady-state interactive UI; <100 ms perceived touch latency in 95% of interactions (SC-005); <3 s warm-cache cold start on a representative mid-tier mobile device (SC-008); MNA solve <50 ms p95 at MVP circuit size (≤10 nodes); CI fails if total JS bundle exceeds **250 KB gzipped**.
**Constraints**: Offline-capable after first load (Principle III); no required network calls; 2D / 2.5D Canvas only (no WebGL, no 3D); touch-first input; simulator readings MUST come from the MNA solver, not lookup tables (Principle I).
**Scale/Scope**: MVP = 1 circuit (≤10 nodes, 1 correct + 1 distractor part in inventory). v1.x roadmap extends to roughly 5–10 circuits with audio path and additional component states.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluated against `.specify/memory/constitution.md` v1.0.0.

| # | Principle | Gate | Pre-research | Post-design |
|---|---|---|---|---|
| I | Physical Accuracy via MNA (NON-NEGOTIABLE) | Solver MUST be MNA-based; readings MUST derive from solve output, not hand-tuned values. | ✓ Plan commits to a custom MNA solver; FR-001 ±5%/±10 mV tolerance gives a measurable correctness target. | ✓ `data-model.md` Component shape and `contracts/circuit-template.schema.json` provide the conductance + current inputs the MNA matrix needs; no shortcut tables anywhere. |
| II | Mobile-First Performance | Mobile perf budgets defined and measurable; solver event-driven; bundle size capped. | ✓ 60 fps / <100 ms touch / <3 s load / MNA <50 ms p95 / <250 KB bundle, all defined. Solve runs only on probe or component change. | ✓ Vanilla-TS choice (research R1) keeps bundle in budget; perf CI check (research R7) enforces the cap. |
| III | Offline-First PWA | No required network calls; persistence is client-side; degrades gracefully if offline. | ✓ Vite PWA plugin + IndexedDB; no backend. | ✓ No API contracts introduced. Service worker precaches all build outputs; runtime falls back to cache for asset requests. `quickstart.md` includes an offline manual-test step. |
| IV | Curated Circuit Templates (no SPICE) | No general-purpose schematic editor in scope; circuits ship as JSON templates. | ✓ Circuit Template is a read-only JSON contract; no editor surface in source tree. | ✓ `contracts/circuit-template.schema.json` defines the curated authoring format. The runtime treats templates as read-only. |
| V | Real-Time Sensory Feedback Loop | Visual feedback for every meaningful action; audio reflects solver state when audio ships. | ✓ for MVP scope (visual feedback only — audio deferred per Q1 clarification). Probe placement, component remove/place, and successful repair all surface visual feedback. Audio gates re-checked when post-MVP audio circuit ships. | ✓ Same. `data-model.md` `audioOutputNode` field is `null` for MVP and becomes non-null when an audio circuit ships, gating audio implementation work. |

**Result:** PASS. No violations; Complexity Tracking section is intentionally empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-vintage-amp-repair-sim/
├── spec.md                              # /speckit.specify + /speckit.clarify (committed)
├── plan.md                              # This file (/speckit.plan)
├── research.md                          # Phase 0 — tech decisions + rationale
├── data-model.md                        # Phase 1 — entity schemas
├── contracts/
│   ├── circuit-template.schema.json     # JSON-Schema for curated circuit JSON
│   └── persistence-schema.md            # IndexedDB stores, keys, migration policy
├── quickstart.md                        # Phase 1 — how to develop, test, validate the MVP
└── checklists/
    └── requirements.md                  # /speckit.specify quality checklist (committed)
```

### Source Code (repository root)

```text
src/
├── solver/                  # MNA solver — pure functions, zero DOM
│   ├── matrix.ts            # dense matrix ops (LU decomposition for solve)
│   ├── mna.ts               # build [G][V] = [I] from circuit, solve for V
│   ├── stamping.ts          # per-component conductance/current contributions
│   └── index.ts
├── circuits/                # Curated circuit templates (data + types)
│   ├── 001-voltage-divider.json
│   ├── schema.ts            # TS types matching contracts/circuit-template.schema.json
│   └── load.ts              # JSON validation + hydration
├── components/              # Component model + failure-mode behavior
│   ├── component.ts
│   └── modes.ts             # nominal | shorted (open + degraded deferred)
├── ui/
│   ├── schematic.ts         # Canvas: schematic view + test-point annotations
│   ├── pcb.ts               # Canvas: PCB top-down view
│   ├── multimeter.ts        # Probe placement + reading display (DOM overlay)
│   ├── inventory.ts         # Inventory drawer (DOM overlay)
│   ├── feedback.ts          # Visual feedback (probe ping, place success, etc.)
│   └── touch.ts             # Pointer-event hit-testing against circuit data
├── inventory/               # Parts inventory state machine
│   └── inventory.ts
├── session/                 # Repair session state + transitions
│   └── repair-session.ts
├── persistence/             # IndexedDB layer (via idb)
│   └── store.ts
├── pwa/                     # Service worker registration helpers
│   └── register.ts
└── main.ts                  # App bootstrap

tests/
├── unit/                    # Vitest, no browser
│   ├── solver/              # MNA correctness vs analytical voltage divider
│   ├── components/
│   ├── session/
│   └── inventory/
├── integration/             # Vitest + happy-dom
│   ├── repair-flow.spec.ts  # diagnose → remove → place → verify
│   └── persistence.spec.ts  # state survives reload
└── e2e/                     # Playwright, real browser
    ├── diagnose.e2e.ts      # MVP touch flow on a phone-sized viewport
    └── offline-pwa.e2e.ts   # install + airplane-mode regression

public/
├── manifest.webmanifest
└── icons/                   # PWA icon set

index.html
vite.config.ts
tsconfig.json
package.json
.github/workflows/ci.yml     # type-check, vitest, playwright, bundle-size check
```

**Structure Decision**: Single project, frontend-only PWA (Option 1 from the template, adapted). The solver is isolated as pure functions so it can be unit-tested without a DOM. `circuits/` holds curated templates as static JSON (Principle IV); no runtime authoring surface exists. Tests are split into:

- **unit** (Vitest, no browser) — solver correctness against analytical references, component-mode logic, session-state transitions, inventory math.
- **integration** (Vitest + happy-dom) — full diagnose → repair → verify flow, and persistence-survives-reload.
- **e2e** (Playwright, real browser) — touch input on a phone-sized viewport, PWA install, airplane-mode functionality.

No backend, so no `api/` or `backend/` tree. No `mobile/`, since iOS/Android delivery is via the PWA, not a native shell.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified.**

No violations. Section intentionally empty.
