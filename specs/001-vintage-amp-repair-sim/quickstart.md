# Quickstart: VARS — MVP

How to develop, test, and validate the Vintage Amp Repair Sim MVP locally. This document is paired with `plan.md` and assumes Phase 0 / Phase 1 decisions hold.

---

## Prerequisites

- **Node.js** 20+ (LTS).
- **npm** 10+ (or pnpm / yarn — the lockfile we ship is npm's, but the project doesn't depend on workspace features).
- A modern browser for manual testing — Chrome, Safari, or Firefox.
- For mobile validation: an iPhone (Safari iOS 16+) or Android device (Chrome) on the same Wi-Fi as the dev machine, or a real-device cloud (BrowserStack / Sauce) for the bundle / PWA / offline checks.

---

## First-time setup

```bash
# from repo root
npm install
```

That's it. No DB to provision, no environment variables, no auth keys. The app is offline-first by design.

---

## Common tasks

### Run the dev server

```bash
npm run dev
```

Vite serves on `http://localhost:5173` with HMR. To test on a phone over LAN, use `npm run dev -- --host` and visit `http://<your-lan-ip>:5173` from the device.

### Build for production

```bash
npm run build
```

Outputs to `dist/`. The build emits a Workbox-generated service worker and a precache manifest of every shipped asset (Constitution Principle III).

### Preview the production build

```bash
npm run preview
```

Serves `dist/` on `http://localhost:4173`. Use this for manual offline-mode validation (see below).

### Type-check

```bash
npm run typecheck    # runs `tsc --noEmit`
```

### Unit tests (Vitest, no browser)

```bash
npm run test:unit
```

Covers `src/solver/`, `src/components/`, `src/session/`, `src/inventory/`. Should run in <2 s.

The MNA solver tests are the most important: they run a known-analytical voltage divider through the solver and assert the computed node voltages match the analytical answer within the FR-001 tolerance (±5% / ±10 mV, whichever is larger). If these fail, do **not** ship.

### Integration tests (Vitest + happy-dom)

```bash
npm run test:integration
```

Exercises end-to-end repair flows without a real browser. Includes the persistence-survives-reload test (FR-012).

### End-to-end tests (Playwright, real browser)

```bash
npx playwright install --with-deps    # one-time
npm run test:e2e
```

Covers:

- Touch-driven probe placement and component remove/place flows on a phone-sized viewport.
- PWA install + airplane-mode (`context.setOffline(true)`) regression — verifies the app stays fully functional with no network.

### Bundle-size check

```bash
npm run check:bundle
```

Reads the Vite manifest after a production build and fails if the gzipped JS bundle exceeds **250 KB** (Constitution Principle II budget). Wired into CI.

---

## Manual MVP validation

These checks are run before tagging a release. Most are also automated, but a human walkthrough catches things tests don't.

### Story 1 — Diagnose

1. `npm run dev` and load the page on a phone-sized viewport (Chrome DevTools → Pixel 7 or actual device).
2. Confirm the MVP voltage-divider circuit loads with:
   - Schematic visible
   - Test-point annotations showing `TP1: +12.0 V`, `TP2: +6.0 V`, `TP3: 0 V (GND)` (or whatever the curated values are)
   - PCB view rendered
   - Multimeter ready
3. Place probes on TP1 (V+ rail) and GND. Confirm reading reads ~12 V (within FR-001 tolerance).
4. Place probes on TP2 (mid-divider) and GND. Confirm reading is ~0 V (since one resistor is shorted — the divider collapses).
5. Confirm: the *expected* value from the schematic annotation is +6 V, the *observed* value is ~0 V, and the discrepancy points the learner at the shorted component.

### Story 2 — Repair + Verify

1. Long-press the shorted resistor on the PCB view to remove it. Confirm the component visibly leaves the board and the simulation now reports a floating node at TP2 (multimeter shows "OL").
2. Open the parts inventory drawer. Confirm exactly two entries: one correct-value resistor + one wrong-value distractor.
3. **Wrong-value path (US2 AS3)**: Drag the wrong-value distractor onto the empty position. Re-probe TP2 vs GND. Confirm reading is *not* +6 V (it reflects the wrong value's behavior, not the correct one's).
4. Remove the wrong part. Place the correct-value resistor.
5. Re-probe TP2 vs GND. Confirm reading is +6 V within tolerance and the session transitions to `repaired`. Visual feedback (success indicator) fires.

### Persistence (FR-012)

1. Mid-repair (after removing but before placing), close the browser tab.
2. Reopen the page. Confirm:
   - The same circuit is loaded.
   - The shorted resistor is still removed (not re-spawned).
   - The probe positions and inventory state match what was on screen at close.

### Offline (FR-009, Constitution Principle III)

1. `npm run build && npm run preview`.
2. Load `http://localhost:4173` in a clean browser profile. Walk through Story 1 to warm the cache.
3. Stop the preview server (`Ctrl+C`).
4. Reload the page. Confirm: the app still loads, the circuit still works, probing still produces correct readings, repair flow still works, persistence still works. **Nothing should fail because the network is gone.**
5. (Real-phone version): repeat steps 1–2, then put the phone in airplane mode. Confirm same behavior.

---

## Project layout map

```
src/solver/      MNA solver — pure functions. Start here when debugging readings.
src/circuits/    Curated circuit JSON. Edit here to author new circuits.
src/components/  Component model + failure-mode behavior.
src/ui/          Canvas rendering, multimeter, inventory drawer, touch handling.
src/session/     Repair-session state machine + transitions.
src/persistence/ IndexedDB layer.
src/pwa/         Service-worker registration helpers.
tests/unit/      Vitest, no browser.
tests/integration/  Vitest + happy-dom.
tests/e2e/       Playwright.
public/          PWA manifest + icons.
```

---

## CI expectations

The CI pipeline (`.github/workflows/ci.yml`, to be added in tasks) runs:

1. `npm ci`
2. `npm run typecheck`
3. `npm run test:unit`
4. `npm run test:integration`
5. `npm run build`
6. `npm run check:bundle`
7. `npm run test:e2e` (Playwright in headless Chromium + WebKit)

A merge to `master` requires all of the above to pass. Bundle-size regressions are blocking (Constitution Principle II is operational only if the budget is enforced).

---

## When something doesn't match the spec

- If a probe reading disagrees with the analytical answer by more than the FR-001 tolerance, **the solver is wrong** — fix `src/solver/` before changing tolerances or test fixtures.
- If a repair "succeeds" but the readings don't actually match a healthy circuit (or vice versa), check `RepairSession`'s `in-progress → repaired` transition logic in `src/session/repair-session.ts`.
- If the app fails offline after first load, the service worker isn't precaching correctly — inspect `vite.config.ts`'s `vite-plugin-pwa` configuration first.

---

## Cross-references

- `spec.md` — what the system is supposed to do (user-facing).
- `plan.md` — the technical plan (this document's index).
- `research.md` — why each technical choice was made.
- `data-model.md` — entity shapes.
- `contracts/circuit-template.schema.json` — circuit-template authoring contract.
- `contracts/persistence-schema.md` — IndexedDB on-disk contract.
- `.specify/memory/constitution.md` — project-wide non-negotiables.
