# Phase 1 Data Model: VARS — MVP

Entity schemas and state transitions derived from `spec.md` Key Entities and Functional Requirements. TypeScript-style interfaces; the canonical JSON-Schema for **Circuit Template** is in `contracts/circuit-template.schema.json`.

All entities live in the client. There is no server (Constitution Principle III).

---

## Component

```ts
type ComponentType =
  | 'resistor'
  | 'capacitor'
  | 'inductor'
  | 'voltage-source'

type FailureMode = 'nominal' | 'shorted' | 'open' | 'degraded'

interface Component {
  id: string                              // unique within a CircuitTemplate
  type: ComponentType
  nominalValue: number                    // SI units (Ω, F, H, V)
  actualValue: number                     // simulated; may diverge from nominal via failure mode or wrong-value placement
  failureMode: FailureMode
  position: { x: number; y: number }      // PCB-view coordinates (px in template's local coord space)
  nodes: [string, string]                 // exactly two nodes the component connects (MVP: 2-terminal only)
}
```

**Validation rules** (FR-003, FR-005):

- `actualValue > 0` for `resistor`, `capacitor`, `inductor`.
- For `failureMode === 'shorted'`, the solver treats conductance as `1 / R_short` where `R_short = 1e-6 Ω` (large but finite — avoids singular matrices in MNA).
- v1 implements behavior **only** for `failureMode ∈ { 'nominal', 'shorted' }` (Q3 clarification). The data model accepts the others; runtime treats unknown modes as `nominal` and emits a console warning.

**State transitions** (in-game actions that mutate `failureMode` or `actualValue`):

- `nominal → shorted`: only as authored — components don't fail mid-session in MVP.
- `(any) → nominal`: a learner removes a faulty component and places a healthy replacement. Implemented as: the failed `Component` is removed from the active circuit; a new `Component` (constructed from an `InventoryEntry`) takes its `position`. The original `Component.id` is replaced.

---

## CircuitTemplate

```ts
interface CircuitTemplate {
  id: string                              // e.g., "001-voltage-divider"
  schemaVersion: 1                        // bumps for breaking JSON shape changes
  displayName: string                     // e.g., "Voltage Divider Tutorial"
  difficulty: 'basic' | 'intermediate' | 'advanced'
  nodes: string[]                         // node IDs; MUST include 'GND'
  components: Component[]                 // initial state, may include shorted/etc.
  testPoints: TestPoint[]                 // FR-014: annotated reference panel
  audioOutputNode: string | null          // null = no audio path (MVP); string = node ID feeding Web Audio
  inventory: InventoryConfig              // pre-populated parts for this circuit
  expectedHealthyComponents: ExpectedComponent[]  // canonical "what a healthy circuit looks like" — used to verify repair
}

interface TestPoint {
  id: string                              // e.g., "TP1"
  label: string                           // visible label on schematic
  node: string                            // node ID this test point taps
  expectedHealthyVoltage: number          // volts (FR-014)
  hint?: string                           // optional learner-facing description
}

interface InventoryConfig {
  entries: InventoryEntry[]
}

interface InventoryEntry {
  componentType: ComponentType
  value: number
  count: number
}

interface ExpectedComponent {
  // For each board position, what value/type a healthy circuit has.
  // Used by the repair-verification check after the learner re-probes.
  positionId: string                      // matches Component.id in the original (broken) state
  type: ComponentType
  expectedValue: number                   // SI units
}
```

**Validation rules** (enforced at load time via `zod`, see research R10):

- Exactly one node MUST be named `'GND'`. The MNA solver uses `GND` as the reference node.
- Every `testPoints[i].node` MUST appear in `nodes`.
- Every `components[i].nodes[*]` MUST appear in `nodes`.
- For the MVP circuit, `inventory.entries` MUST contain **exactly one** entry whose `(type, value)` matches the failed component's healthy value, and **exactly one** entry of the same `type` with a different `value` (the wrong-value distractor) — Q5 clarification.
- `audioOutputNode` MUST be `null` for any circuit that ships before the audio path lands; circuits with audio MUST also pass the post-MVP audio gate (Constitution Principle V).

---

## ProbeReading (transient — not persisted)

```ts
interface ProbeReading {
  nodeA: string
  nodeB: string
  voltage: number                         // V_A − V_B; sign matters
  isFloating: boolean                     // true if either node is electrically isolated (no path to GND)
  computedAt: number                      // monotonic ms timestamp
}
```

**Notes**:

- Recomputed on demand by the MNA solver when the learner places probes or any component changes.
- `isFloating` is the surface affordance for the **Floating-Node** edge case in the spec — when probing isolated nodes, the UI shows "OL" (open-loop) rather than a misleading numeric voltage.

---

## RepairSession

```ts
type RepairSessionStatus = 'in-progress' | 'repaired' | 'abandoned'

interface RepairSession {
  id: string                              // uuid
  circuitId: string                       // CircuitTemplate.id
  startedAt: number
  status: RepairSessionStatus
  // mutable per-session state
  componentOverrides: Record<string, Component>   // by Component.id; tracks removals + replacements
  inventoryRemaining: InventoryConfig             // decreases as user places parts
  cleaningProgress: Record<string, number>        // pot id → 0..1 (post-MVP, Story 3)
  history: RepairEvent[]                          // for replay / undo (post-MVP)
}

interface RepairEvent {
  at: number
  kind: 'probe' | 'remove' | 'place'
  payload:
    | { kind: 'probe'; reading: ProbeReading }
    | { kind: 'remove'; componentId: string }
    | { kind: 'place'; positionId: string; from: InventoryEntry }
}
```

**State transitions** (FR-004, FR-012, US2 acceptance):

- `in-progress → repaired`: triggered when *all* `testPoints[*]` re-read within the FR-001 tolerance (±5% relative or ±10 mV absolute, whichever is larger) of their `expectedHealthyVoltage`. The check runs after every successful `place` event.
- `in-progress → abandoned`: explicit user action (close, switch circuit). The session is preserved in IndexedDB and resumable on re-open (FR-012).
- `repaired → in-progress`: **not supported.** Once repaired, the session is closed; restarting the circuit creates a new session.

**Persistence** (FR-008, FR-012):

- Every state change writes the session synchronously to IndexedDB before acknowledging the action.
- On boot, the most recent `in-progress` session for the active circuit is auto-resumed.

---

## PartsInventoryAccount (cross-session)

```ts
interface PartsInventoryAccount {
  ownerKey: string                        // local-only identity; opaque
  pool: InventoryEntry[]                  // accumulated across all completed sessions
  schemaVersion: 1
}
```

**Notes**:

- Per Constitution Principle III, this is local-only. No server persistence, no PII.
- `ownerKey` is generated on first launch (`crypto.randomUUID()`) and stored in IndexedDB; never leaves the device.
- The MVP doesn't grow the pool — the inventory is configured per-circuit. The cross-session pool exists in the model so post-MVP gameplay (earning parts via repairs) has a place to land without schema churn.

---

## Persistence stores (IndexedDB layout)

See `contracts/persistence-schema.md` for the on-disk contract. Summary:

| Object store | Key | Value |
|---|---|---|
| `sessions` | `RepairSession.id` | `RepairSession` |
| `inventory` | constant key `'me'` | `PartsInventoryAccount` |
| `meta` | `'schemaVersion'` | `number` (whole-DB version) |

---

## Cross-references

- **FR-001** (tolerance) → enforced in `RepairSession` state-transition check (`in-progress → repaired`).
- **FR-003** (component states) → `FailureMode` union; v1 implements `nominal`, `shorted`.
- **FR-004** (remove + replace) → `RepairEvent` payload variants; updates `componentOverrides`.
- **FR-005** (no faked readings) → solver always reads from `Component.actualValue`, never from a lookup of the "intended" value.
- **FR-008, FR-009, FR-012** (offline persistence + resume) → IndexedDB stores; `RepairSession.status === 'in-progress'` survives reload.
- **FR-014** (annotated reference panel) → `TestPoint.expectedHealthyVoltage`.
