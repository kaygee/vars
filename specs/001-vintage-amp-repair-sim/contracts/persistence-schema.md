# Persistence Schema Contract — VARS IndexedDB Layout

This is the on-disk contract for VARS client-side persistence. Per Constitution Principle III, all state lives on the learner's device. This document is the source of truth for IndexedDB store layout, key conventions, and migration policy.

**Database name**: `vars`
**Database version**: `1` (IndexedDB-level version; bumps when stores are added/removed)

---

## Object stores

### `sessions`

Holds every `RepairSession` the learner has started, including in-progress and completed.

| Field | Type | Notes |
|---|---|---|
| **Key path** | `id` | UUID generated client-side (`crypto.randomUUID()`) |
| **Indexes** | `by_circuit` on `circuitId` | Find all sessions for a given circuit |
|  | `by_status` on `status` | Find all `in-progress` sessions for resume |
| **Value shape** | `RepairSession` | See `data-model.md` |

Boot behavior: on app start, query `by_status === 'in-progress'`, take the most recent (max `startedAt`), and auto-resume.

### `inventory`

Holds the cross-session parts pool.

| Field | Type | Notes |
|---|---|---|
| **Key path** | n/a (out-of-line key) | Single record at fixed key `'me'` |
| **Indexes** | none | |
| **Value shape** | `PartsInventoryAccount` | See `data-model.md` |

The `'me'` key is intentional: this is single-tenant local state. There is no multi-account model and there is no path to introduce one without violating Principle III (no backend = no central account system).

### `meta`

Misc client metadata.

| Key | Value type | Purpose |
|---|---|---|
| `'schemaVersion'` | `number` | Whole-DB logical version (separate from IndexedDB-level version) |
| `'firstLaunchAt'` | `number` (ms epoch) | When the learner first opened VARS on this device |
| `'lastResumedSession'` | `string \| null` | Most recently resumed `RepairSession.id`; speeds up boot |

---

## Migration policy

Two layers of versioning, on purpose:

1. **IndexedDB version** (`indexedDB.open('vars', N)`) — bumps when **stores are added or removed** or their indexes change. Migrations run in `onupgradeneeded`.
2. **Logical schema version** (the `schemaVersion: number` field on each persisted record + the `meta.schemaVersion` whole-DB value) — bumps when **record shapes change** without store-level surgery.

### Forward-compatibility promise

- The runtime **MUST** refuse to load a record whose `schemaVersion` is higher than the running build understands. Surface a clear error to the user ("This save was made with a newer version of VARS; please update.") rather than corrupting it.
- The runtime **MUST** be able to load any record from a previous `schemaVersion` and migrate it on read (lazy migration). Eager migration on `onupgradeneeded` is acceptable when cheap.

### v1 surface

- IndexedDB version: `1`.
- All record-level `schemaVersion`: `1`.
- No migration logic shipped (this is the baseline).

---

## Failure modes

- **IndexedDB unavailable** (private-mode Safari historically): the app MUST surface a clear "VARS needs storage to save your progress" message and refuse to start a repair session. We will *not* fall back to LocalStorage — its 5 MB cap and synchronous I/O conflict with future scale, and the failure mode hides bugs.
- **Quota exceeded**: surface a "Storage full" error and refuse the write. Do not silently drop state.
- **Concurrent tabs**: IndexedDB handles concurrent access; sessions are written transactionally (one transaction per state change). Two open tabs of the same session is a supported edge case — last-write-wins; the second tab gets a `versionchange` event and reloads.

---

## What's *not* in IndexedDB

- **Circuit templates** (`circuits/*.json`) — shipped as static assets, served by the service worker from precache. Templates are read-only and never mutate at runtime.
- **App shell** (HTML/CSS/JS/icons) — Workbox precache via `vite-plugin-pwa`.
- **Audio buffers / DSP state** — runtime-only; never persisted.

---

## Cross-references

- `data-model.md` — entity shapes (`RepairSession`, `PartsInventoryAccount`)
- `research.md` R5 (PWA / service worker), R9 (schema versioning)
- Constitution Principle III (Offline-First PWA)
- Spec FR-008, FR-009, FR-012
