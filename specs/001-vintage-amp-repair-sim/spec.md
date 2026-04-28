# Feature Specification: Vintage Amp Repair Sim (VARS)

**Feature Branch**: `master` (path-B in-place restructure; no dedicated feature branch)
**Created**: 2026-04-28
**Status**: Draft
**Input**: User description: "Web-based educational game that teaches the fundamentals of 1970s-era discrete electronics repair by simulating real circuit behavior, optimized for mobile, with diagnostic, repair, and sensory-feedback gameplay."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Diagnose a Broken Amp by Probing (Priority: P1)

A learner opens VARS on their phone, selects a broken vintage amplifier, and uses a virtual multimeter to probe points in the circuit. By comparing the readings they observe to the readings they would expect from a working amp, they identify which component is faulty.

**Why this priority**: This is the heart of the educational loop and the MVP. Without trustworthy probe readings derived from accurate circuit simulation, no other gameplay matters — the entire premise of "see the physical layer" depends on this story working end-to-end. It also exercises the simulation foundation that every later story builds on.

**Independent Test**: Can be fully tested by loading a single broken-circuit template (e.g., a voltage divider with one shorted component), having the learner probe two specified nodes, and verifying the displayed voltage matches the expected value computed by the underlying simulation within an acceptable tolerance. Delivers value as a standalone "diagnose-only" experience even before any repair gameplay exists.

**Acceptance Scenarios**:

1. **Given** a learner has loaded a known-broken circuit template, **When** they place virtual probes on two nodes of the circuit, **Then** the multimeter displays the potential difference between those nodes consistent with the circuit's true simulated state.
2. **Given** the same broken-circuit template loaded twice in two separate sessions, **When** the learner probes the same two nodes both times, **Then** they observe the same reading (deterministic behavior).
3. **Given** a learner probes the *same* node with both leads, **When** they look at the multimeter, **Then** they see a reading of zero (no potential difference).
4. **Given** a learner probes a node that is electrically isolated (e.g., a component has been removed leaving a floating node), **When** they look at the multimeter, **Then** the reading reflects an open-circuit condition rather than a misleading numeric value.

---

### User Story 2 - Replace a Faulty Component and Verify the Repair (Priority: P2)

After diagnosing which component is broken, the learner removes that component from the circuit board view, selects a replacement from their parts inventory, and places it in the empty position. They re-probe the circuit and confirm the readings now match the expected behavior of a working amp.

**Why this priority**: Diagnosis without repair is incomplete as a *learning* experience — the loop closes only when the learner can act on their hypothesis and see the consequence. This story turns the simulator into a game by adding agency and a success state. It depends on Story 1 because verification requires probing, but it does not depend on Story 3.

**Independent Test**: Can be tested by giving the learner a circuit with a known fault and a parts inventory containing the correct replacement, having them desolder the broken part and place the replacement, and verifying that re-probing the circuit returns readings consistent with a healthy circuit. Delivers a complete diagnose-fix-verify loop on its own.

**Acceptance Scenarios**:

1. **Given** a circuit with a clearly faulty component identified during Story 1, **When** the learner performs the "remove" interaction on that component, **Then** the component is taken off the board and the circuit's simulated state updates to reflect its absence.
2. **Given** the learner has removed a component and has at least one matching replacement in their parts inventory, **When** they place the replacement on the now-empty position, **Then** the circuit's simulated state updates to reflect the new component values.
3. **Given** the learner places a replacement of the *wrong* value (e.g., wrong resistance), **When** they re-probe the circuit, **Then** the readings reflect the wrong-value behavior — no readings are faked or hidden.
4. **Given** the learner has correctly replaced the failed component, **When** they re-probe the circuit at the diagnostic test points, **Then** all readings now match the expected values of a healthy amp within tolerance, and the session is marked as a successful repair.

---

### User Story 3 - Hear and Feel the Amp Through Repair (Priority: P3)

Throughout diagnosis and repair, the learner can hear the amplifier's output through their device. A broken amp sounds wrong (hum, distortion, silence, scratchy); as components are replaced or cleaned, the audio behavior changes accordingly. For variable controls (e.g., a noisy volume knob), the learner can scrub a virtual contact-cleaner pad against the control to reduce the scratchiness over repeated strokes.

**Why this priority**: The sensory layer is what binds abstract circuit values to embodied learning. Without it, VARS degrades to a logic puzzle. But it is correctly a P3 story because the educational loop in Stories 1 and 2 is meaningful even silently, and audio interactions add the most engineering risk per unit of pedagogical value. Ship the diagnostic and repair loops first, then layer this on.

**Independent Test**: Can be tested with a single circuit that has at least one degradation mode audibly different from nominal (e.g., a leaky capacitor that low-passes the output, or a scratchy potentiometer), confirming that the listener can perceive the difference and that the perceived behavior changes appropriately when the component is repaired or cleaned. Delivers the "feel" of vintage gear even on a single circuit.

**Acceptance Scenarios**:

1. **Given** a healthy circuit with audio enabled, **When** the learner triggers the amp's output, **Then** they hear clean, undistorted audio.
2. **Given** a circuit with a degraded ("leaky") component in the signal path, **When** the learner triggers output, **Then** the audio reflects that degradation in a way perceptibly distinct from the healthy case (e.g., dulled high frequencies, distortion, or hum).
3. **Given** a "scratchy" potentiometer in the circuit, **When** the learner moves the control with no cleaning, **Then** moving it produces noisy crackle audible at the output.
4. **Given** a scratchy potentiometer, **When** the learner repeatedly applies a virtual contact-cleaner action across the control, **Then** the noisy crackle perceptibly diminishes; after sufficient cleaning the control behaves smoothly.
5. **Given** the learner has muted the device or denied audio permission, **When** they attempt the diagnose-and-repair loop, **Then** Stories 1 and 2 still function fully without audio.

---

### Edge Cases

- **Floating nodes**: When a component is removed mid-repair leaving a portion of the circuit electrically isolated, probe readings on the isolated portion must indicate an open-circuit condition rather than display a confusing numeric voltage.
- **Wrong-value replacements**: If the learner installs a component whose value differs from the original spec, the circuit must behave according to physics, not according to what the learner intended — wrong values produce wrong readings.
- **Same-node probing**: Probing the same node with both leads must read zero, not raise an error or display undefined behavior.
- **Touch precision on dense boards**: Components on a phone screen are visually small; the system must distinguish "tap component A" from "tap component B" reliably even when they are spatially close.
- **Mid-session interruption**: A learner may close the browser, lock their phone, or lose focus mid-repair; their progress on the current circuit must persist and resume cleanly without corruption.
- **No audio device or denied permission**: If audio is unavailable, Story 3 features degrade silently but the diagnostic and repair loops must continue uninterrupted.
- **Empty inventory**: If the learner attempts a repair without having a suitable replacement part available, the system must communicate this clearly rather than allowing a non-physical "place from nothing" action.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST simulate the electrical behavior of pre-authored circuit templates with sufficient fidelity that observed voltages and currents match physically correct values within a documented tolerance.
- **FR-002**: System MUST allow learners to place virtual probes on any junction in the visible circuit and display the potential difference between two probed points.
- **FR-003**: System MUST support at least four component states across all simulated parts: nominal, open (failed-disconnected), shorted (failed-connected), and degraded (e.g., leaky, drifted from nominal value).
- **FR-004**: System MUST allow learners to remove a component from a circuit and replace it with another component drawn from a parts inventory, with the simulated circuit state updating to reflect each change.
- **FR-005**: System MUST behave according to the placed components' values, even when those values are wrong for the circuit — no falsified readings, no hidden corrections.
- **FR-006**: System MUST produce audible output reflecting the simulated state of the circuit's output node, including audibly distinct symptoms when components are degraded.
- **FR-007**: System MUST allow learners to apply a "contact cleaner" action to variable controls (potentiometers) via repeated touch motion, with noise behavior diminishing as cleaning accumulates.
- **FR-008**: System MUST persist learner progress (active session, completed repairs, parts inventory) locally on the learner's device.
- **FR-009**: System MUST function fully without an internet connection after first load, including all gameplay (probe, repair, audio, cleaning) and progress persistence.
- **FR-010**: System MUST be optimized for touch input on mobile devices, with all interactions (probe placement, component removal, component placement, control scrubbing) performable using only touch.
- **FR-011**: System MUST present circuits as a curated set of pre-authored templates; the system MUST NOT expose a general-purpose schematic-authoring surface to learners.
- **FR-012**: System MUST resume cleanly from interruptions (browser close, tab change, device lock) with no corruption of in-progress repair state.
- **FR-013**: System MUST communicate clearly when a learner attempts an action that is not currently possible (e.g., placing a component without a matching part in inventory) rather than silently failing.

### Key Entities

- **Circuit Template**: A pre-authored circuit with a defined topology, a defined set of components in known nominal/failed/degraded states, an expected "healthy" behavior used as the reference for verification, and an associated audio output point. Curated, not learner-authored.
- **Component**: An element in a circuit. Has an identity, a nominal value, an actual value (which may drift from nominal due to age or failure), a failure-mode state (`nominal | open | shorted | leaky` and similar), and a spatial position on the circuit board view.
- **Probe Reading**: A measurement of potential difference between two specified points in a circuit, derived from the live simulation. Not stored as a static value.
- **Repair Session**: A learner's in-progress engagement with a single Circuit Template — which components have been removed, what has been replaced, what has been cleaned, current inventory, and a status (in-progress / repaired / abandoned).
- **Parts Inventory**: The collection of replacement components available to the learner, drawn from across sessions. Each entry has a part type, a value, and a count.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: After completing one tutorial session, at least 70% of new learners can correctly identify the failed component in a never-before-seen broken-circuit template using only the multimeter, without referring to outside material.
- **SC-002**: At least 80% of learners who successfully diagnose a faulty component can also complete the corresponding repair (remove + replace + verify) on their first attempt within a single session.
- **SC-003**: Learners with prior electronics experience report that the audible behavior of degraded components is perceptibly accurate — at least 75% rate Story 3's audio feedback as "matches what I'd expect from real gear" or stronger.
- **SC-004**: Learners can complete a full diagnose-repair-verify loop on the MVP circuit in under 10 minutes on their first successful attempt.
- **SC-005**: Probe interactions, component placement, and control scrubbing feel responsive on mid-tier mobile devices — perceived latency between touch action and visual or audible response is under 100 ms in 95% of interactions.
- **SC-006**: The application functions identically with the device in airplane mode after first load, including audio, persistence, and all repair gameplay (no degraded modes due to missing network).
- **SC-007**: Learner progress survives at least one full close-and-reopen of the browser/tab without loss of in-progress repair state, in 100% of test sessions.
- **SC-008**: The MVP circuit (Story 1 only) loads and becomes interactive on a representative mid-tier mobile device in under 3 seconds from first launch on warm cache.

## Assumptions

- The primary target audience is learners with a computer-science background or equivalent analytical familiarity who want to understand the physical layer beneath the abstractions they already know. Total beginners are a stretch goal, not the MVP audience.
- Mobile-first means phones and small tablets are the design center; desktop browsers will work but are not the optimization target.
- The initial release ships with a small number of curated circuit templates; one fully fleshed-out circuit is sufficient to deliver the MVP (Story 1).
- The application is delivered through a standard web browser; learners are not asked to install a native app.
- The application has no required server-side component; all state lives on the learner's device.
- Audio playback requires a working audio output (speaker or headphones); learners without audio retain access to Stories 1 and 2.
- "Mid-tier mobile device" is interpreted as a mainstream phone roughly 2–3 years old at the time of release.
- Monetization (free, paid, ads) is out of scope for this specification.
- Visual fidelity is 2D / 2.5D; high-end 3D rendering is explicitly out of scope.
- Curation of circuit templates is performed by the project maintainers; learner-authored circuits are out of scope.
