<!--
SYNC IMPACT REPORT
==================
Version change: (uninitialized template) → 1.0.0
Bump rationale: Initial ratification of project constitution; no prior version to compare against.

Modified principles:
  - All five principle slots populated for the first time:
    I. Physical Accuracy via Modified Nodal Analysis (NON-NEGOTIABLE)
    II. Mobile-First Performance
    III. Offline-First Progressive Web App
    IV. Curated Circuit Templates (Not a Generic SPICE)
    V. Real-Time Sensory Feedback Loop

Added sections:
  - Core Principles (5 principles)
  - Technical Constraints & Performance Standards
  - Development Workflow & Quality Gates
  - Governance

Removed sections:
  - None (template placeholders only)

Templates requiring updates:
  - .specify/templates/plan-template.md       ⚠ pending — "Constitution Check" gate is currently a placeholder ("[Gates determined based on constitution file]"); a future plan run should enumerate gates derived from Principles I–V here.
  - .specify/templates/spec-template.md       ✅ no change required (no constitution-specific references)
  - .specify/templates/tasks-template.md      ✅ no change required (no constitution-specific references)
  - .specify/templates/checklist-template.md  ✅ not reviewed in detail; no known references requiring update
  - README.md / docs/quickstart.md            n/a — files do not exist in the repository at ratification time

Follow-up TODOs:
  - When the first /speckit.plan runs, populate the plan template's Constitution Check section with explicit gate items derived from Principles I, II, III, and V (e.g., "MNA solver path verified", "mobile perf budget measured", "no required network calls", "audio + visual feedback present for relevant actions").
  - Author/owner field left as VARS Maintainers; replace with named maintainer(s) once ownership is formalized.
-->

# VARS (Vintage Amp Repair Sim) Constitution

## Core Principles

### I. Physical Accuracy via Modified Nodal Analysis (NON-NEGOTIABLE)

Circuit behavior surfaced to the user MUST be derived from a Modified Nodal Analysis (MNA)
solver operating on the directed-graph circuit model. Voltages, currents, and component
responses MUST NOT be faked, hand-tuned, or hardcoded for narrative effect. If a desired
educational outcome cannot be achieved through accurate simulation, the circuit template
itself MUST be redesigned — the solver path is sacrosanct.

**Rationale:** VARS exists to bridge CS abstractions to physical-layer reality. Any
divergence from real electrical behavior defeats the educational mission and would
reduce the project to a themed puzzle game. Solver fidelity is the product.

### II. Mobile-First Performance

Every feature MUST be designed against mid-tier mobile browser budgets first; desktop is
a beneficiary, not a target. Solver invocations MUST be event-driven (on probe or
component change), not continuous, to bound CPU usage. Heavy work (matrix solve >50
nodes, audio DSP) SHOULD be offloaded to Web Workers. Touch interactions (drag, scrub,
long-press) MUST feel responsive (<100ms perceived latency for direct manipulation).

**Rationale:** The target audience interacts on phones. A correct simulator that drains
batteries or stutters on touch fails its users. Mobile constraints discipline the
architecture across the board.

### III. Offline-First Progressive Web App

The application MUST function fully offline as a Progressive Web App after first load.
Game progress, inventory, and user state MUST persist via local storage (or equivalent
client-side persistence). No feature may require a backend service to operate. Optional
cloud features (e.g., sync, leaderboards) MAY be added later but MUST degrade gracefully
to an offline-equivalent experience.

**Rationale:** The educational value is highest in environments with intermittent
connectivity (workshops, classrooms, field). A backend dependency also imposes hosting
cost and operational burden inconsistent with the project's scope.

### IV. Curated Circuit Templates (Not a Generic SPICE)

Circuits MUST ship as pre-defined, hand-curated templates with intentional pedagogical
framing (specific failure modes, specific repair workflows). VARS MUST NOT expose a
generic schematic editor or general-purpose SPICE-like authoring surface to end users.
New circuits enter the system through a deliberate authoring process, not user-generated
content.

**Rationale:** Educational quality, mobile performance, and solver determinism all
benefit from curation. A generic CAD surface would explode scope, performance risk, and
the support matrix without serving the core mission.

### V. Real-Time Sensory Feedback Loop

Every meaningful user action against the circuit (probing, desoldering, replacing,
scrubbing a potentiometer, applying DeoxIT) MUST produce immediate audio and/or visual
feedback. Audio output MUST route through the Web Audio API and reflect the simulated
output node, not a pre-baked sample. Component degradation modes (`leaky`, `shorted`,
`open`, `nominal`) MUST be perceptible in the feedback signal, not merely displayed in
a UI label.

**Rationale:** The "feel" of vintage gear repair — hum, hiss, distortion, the click of a
clean knob — is the dopamine hook that makes the physics memorable. Decoupling feedback
from simulation breaks the educational chain that connects component state to real-world
symptom.

## Technical Constraints & Performance Standards

- **Platform:** Web (mobile-first), delivered as a Progressive Web App. No native
  mobile or desktop binaries.
- **Rendering:** 2D / 2.5D via HTML5 Canvas. 3D rendering is out of scope.
- **Audio:** Web Audio API. Audio path MUST consume simulator output, not loop
  pre-recorded clips for nominal operation.
- **Solver scale:** MNA solver MUST handle circuits of ≥50 nodes within mobile-browser
  performance budgets; circuits exceeding this scale MUST justify the extra cost during
  /speckit.plan and propose a Web Worker or batched-solve strategy.
- **Persistence:** Client-side (LocalStorage / IndexedDB / equivalent). No mandatory
  server-side persistence at any level of the stack.
- **Component model:** All simulated parts MUST extend a common `Component` interface
  exposing at minimum `id`, `nominalValue`, `actualValue`, `failureMode`, and
  `position`. Drift and failure-mode behavior MUST be expressed via this model, not via
  per-component bespoke code paths.
- **Frame budget:** Interactive UI SHOULD hold 60 fps on mid-tier mobile devices during
  steady-state interaction; solver-triggered hitches MUST remain below a 50ms p95
  perceived latency on the same class of device.

## Development Workflow & Quality Gates

- **Spec-Kit flow is the default authoring path:** new features MUST originate as a
  `/specs/<NNN-slug>/spec.md` produced via `/speckit.specify`, planned via
  `/speckit.plan`, and broken down via `/speckit.tasks` before implementation work
  begins. Drive-by implementation without a corresponding spec is discouraged.
- **Constitution Check gate:** every implementation plan MUST pass an explicit
  Constitution Check (see `.specify/templates/plan-template.md`). Violations MUST be
  recorded in the plan's Complexity Tracking table with the simpler alternative that was
  rejected and why.
- **Solver-first design:** for any feature touching circuit behavior, the solver
  contract MUST be defined and validated (against known-good voltage/current outputs for
  reference circuits) before UI work proceeds.
- **Performance evidence:** features that affect the render or solver path MUST be
  validated on at least one mid-tier mobile browser (real device or representative
  emulation) before being considered complete; numbers SHOULD be captured in the
  feature's `quickstart.md` or equivalent artifact.
- **Offline regression check:** any feature merge MUST preserve full offline operation.
  A network-required feature that breaks offline mode is a constitution violation and
  MUST be reverted or gated behind explicit progressive-enhancement.
- **Reviews:** changes touching the solver, the component data model, or the audio
  pipeline MUST receive explicit reviewer sign-off citing Principles I, II, and V
  respectively.

## Governance

This constitution supersedes ad-hoc convention and individual preference. When this
document and other guidance conflict, this document wins until amended.

**Amendment procedure:**

1. Propose the change as a pull request modifying `.specify/memory/constitution.md`.
2. The PR description MUST identify the bump type (MAJOR / MINOR / PATCH) and the
   reasoning, and MUST list templates and docs requiring downstream updates.
3. The PR MUST include any necessary updates to `.specify/templates/*` so dependent
   artifacts stay in sync (Sync Impact Report at the top of this file is the audit
   trail).
4. Approval requires sign-off from the project maintainer(s).
5. On merge, `LAST_AMENDED_DATE` MUST be updated and `CONSTITUTION_VERSION` bumped per
   the policy below.

**Versioning policy (semantic):**

- **MAJOR**: Backward-incompatible governance changes, principle removals, or
  redefinitions that invalidate prior plans.
- **MINOR**: New principle or section added; materially expanded normative guidance.
- **PATCH**: Clarifications, wording, typo fixes, or non-semantic refinements that do
  not change what is required.

**Compliance review:**

- Every `/speckit.plan` invocation runs the Constitution Check gate; violations are
  recorded inline.
- A periodic (≥ quarterly) audit SHOULD walk recent merges against Principles I–V and
  surface drift; findings feed amendments where the constitution lags reality, or
  remediation tasks where the codebase has drifted from the constitution.

**Runtime guidance:** for in-flight, day-to-day development guidance not rising to a
constitutional principle, defer to feature-level `plan.md` and `quickstart.md` artifacts
under `specs/`.

**Version**: 1.0.0 | **Ratified**: 2026-04-28 | **Last Amended**: 2026-04-28
