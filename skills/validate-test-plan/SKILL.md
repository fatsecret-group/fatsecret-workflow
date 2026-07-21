---
name: validate-test-plan
description: Use when a test plan (test-plan.html, a CSV, or any case list) needs to be validated against the implementation — refute-first verification with typed verdicts and evidence, criticality × verifiability routing that shrinks the manual-QA list, and explicit punting of what static reading cannot prove. Backs feature-workflow Step 7a; also runs standalone on any test plan.
---

# Validate Test Plan

## Overview

Verify that a test plan's cases are actually implemented. The core discipline: **"validated" is a claim about evidence, not confidence.** Asking "is this implemented?" invites confirmation bias — a matching function name becomes a PASS — so every case is attacked instead ("find the reason this case is NOT satisfied") and graded with a typed verdict. Static reading is never allowed to impersonate runtime proof.

**Announce at start:** "I'm using the validate-test-plan skill to validate the test plan against the implementation."

## Input

- **The test plan**: `test-plan.html` (from `write-test-plan`), a CSV, or any structured case list. Parse every case into ID/title, preconditions, steps, and its individual **expected-result sub-clauses** — one case usually asserts 3–4 distinct things (a field exists, its value maps correctly, a log line prints, parity with another platform). Sub-clauses are the unit of verification, never the case as a blob.
- **Scope (optional, preferred)**: the feature folder (`design.html` + execution checklist / implementation plan) and/or a git range. Without it (legacy or standalone plans), Phase 1 falls back to code search.

## Phase 0 — Classify every case on two axes

Before verifying anything, tag each case (or sub-clause, where they differ) by **verifiability**:

| Class | Meaning | What counts as proof |
|-------|---------|----------------------|
| **U** — unit-verifiable | deterministic, non-UI logic | a unit test in `Calorie Counter Tests` asserting this clause, run fresh in this session, green |
| **S** — statically traceable | outcome readable from code (payload construction, log format, navigation wiring) | a `file:line` evidence chain per sub-clause: entry point → logic → observable outcome |
| **R** — runtime-only | observable only with the app running (network capture, UI behavior, timing) | a runtime observation (Phase 3), or an explicit punt |
| **X** — externally dependent | cross-platform parity, backend behavior, third-party dashboards | NOT verifiable from this repo — always punted, never graded from local code |

Misclassification is the root failure of naive validation: grading an R/X clause from static reading produces a confident false PASS.

**Time-passage & clock-change trap.** Steps like "let the day/week roll over", "view the streak the next day", "at the start of the new week", or **"change the device date forward/back"** hide a time-dependent sub-clause that a pure function taking `today` as a parameter does NOT prove: the shipped app can show a stale/wrong value because a cached display is re-derived only on data-change hooks (nothing fires when only *time* moves), or because the view reuses a cell across the change. **Split every such case into two sub-clauses:**
- **State/count logic** (does the streak break / carry / recompute correctly as `today` advances) → **U**, but only via a test that drives the app's *real* date-dependent path (view-model foreground/appear handler, seeder, deriver) with `today` injected — see the clock seam in Phase 3 — never a bare pure-function call that already takes `today`.
- **Displayed value across the change** (the dot/tick/label the user actually sees after the rollover or date jump) → **R**: freshness-on-refresh and view identity are runtime/UI, not unit-testable — punt to manual QA on a real device.

Two real misses this guards, both graded VERIFIED(Unit) off a green deriver test yet broken in the app: flexible-streak **T037** (deriver correct, widget kept the old count after a missed day until relaunch) and **sc-62148** (a forward device-date jump painted a phantom tick on an unlogged day while the count stayed correct — a SwiftUI cell-identity residue, invisible to any logic test). The engine being provably clean does NOT clear the on-screen sub-clause.

**Second axis — criticality.** Tag each case **Critical** or **Non-critical**. Critical cases sit on the feature's core path: they touch the design's data-flow spine, implement a D-decision, or their failure breaks the feature's reason to exist. Non-critical cases hang off the edges — rare inputs, cosmetic states, defensive paths. Judge from `design.html` (D-decisions, data-flow diagram), the story-analysis acceptance criteria, and the code itself — not from the case's wording. **Importance and verifiability are orthogonal**: a critical case may be perfectly unit-verifiable, and a non-critical edge case may be runtime-only — neither axis substitutes for the other.

**Human confirmation gate.** Present the criticality assignment as ONE table for a single batch confirmation before verification starts — the model never self-assigns which cases escape rigorous verification. The human's corrections are final.

**Routing** — the two axes together decide who verifies what:

| | Machine-verifiable (U / S / automatable R) | Not machine-verifiable (manual R / X) |
|---|---|---|
| **Critical** | machine verifies with evidence; the human reviews the evidence (spot-check) | **human must verify** — this is the real manual-QA list |
| **Non-critical** | machine verifies; done | **explicit risk acceptance** — waived with human sign-off, honestly recorded, never dressed up as verified |

## Phase 1 — Associate cases with tasks (or code)

- **With workflow context**: map each case to the task that owns it — via the checklist/plan's Unit tests and Files fields and the git history of those files. **A case no task owns is itself a finding** (coverage gap), reported with the story/design item it came from.
- **Standalone**: map each case to the code area that would implement it — cite what the search found, or that nothing was found.

## Phase 2 — Refute-first verification

Verify in batches small enough that the last case gets the same scrutiny as the first (~10 per pass). Read-only verification batches may be delegated per the feature-workflow Delegation section — verifiers return verdicts + citations, never file dumps.

The framing is adversarial: **try to show the case is NOT satisfied.** Per sub-clause, the verdict is one of:

- **VERIFIED** — evidence attached: U → the named unit test + its fresh run output; S → the `file:line` chain
- **PARTIAL** — some sub-clauses verified; names exactly which are not, and why
- **NOT-FOUND** — no implementing code located; the searches tried are listed
- **PUNTED** — R/X: names the runtime step or external confirmation that would verify it (e.g. "Proxyman capture of `trackPurchase` during a promo purchase"; "Android event JSON for the type comparison")

**A bare PASS does not exist.** Uncertainty resolves DOWN (to PARTIAL or PUNTED), never up.

## Phase 3 — Runtime pass (R cases, when tooling allows)

Where the environment has `xcodebuildmcp` + Proxyman MCP: `build_run_sim`, drive the flow (UI automation), and assert on the real artifact — captured request payloads (`get_flows` / `get_flow_detail`), console logs, screenshots. Each successful observation upgrades a PUNTED clause to VERIFIED with the capture cited. **Prioritize critical R cases** — each one automated moves an item off the manual-QA list, which is where this pass pays. R cases that can't be automated stay PUNTED for manual QA; X cases always stay PUNTED.

**Injecting "today" — recover date-dependent *logic* as U without a clock change.** The OS/simulator clock can't be moved without changing the host machine (invasive, off-limits), and the app reads `NSDate` — so "change the date / next day / at week start" cases look runtime-only. But most of the *logic* isn't UI. A DEBUG-only clock seam — FatSecret has `Utils.setTodayDateIntOverrideForTesting` / `clearTodayDateIntOverrideForTesting` (production always falls through to `NSDate`; `getTodayInt`/`getTodayDateInt` honor it) — lets a unit/integration test: set `today` = day N, build the state as of N (log a day, cache the counters the app would have written), advance `today` to N+k, then invoke the app's **real** foreground/refresh path (e.g. `GamiViewModel.fillStreakPastDays` + `streak.sync()`), and assert on the resulting state/count. That turns a date-logic clause from PUNTED(R) into VERIFIED(U) with no clock change and no UI. If no seam exists, adding one is in scope — it is reproduction infrastructure, and it pays for itself the first time a date case needs machine proof. **Hard limit:** the seam proves the *state*, never the *pixels* — a stale tick, a cell that didn't re-render, an animation residue stay **R** for manual/device QA (that gap is exactly how sc-62148 shipped past a green engine).

## Phase 4 — Report

One table: case → criticality → class → verdict → evidence / punt reason → owning task (when mapped). Then, by routing quadrant:

- **Gaps** (NOT-FOUND / PARTIAL, any quadrant): each with what's missing and where it should live
- **Manual-QA list** (critical ∩ not machine-verifiable): what the human must verify by hand — the list this skill exists to shrink
- **Evidence-review list** (critical ∩ machine-verified): the citations / test runs / captures, ready for the human's spot-check
- **Risk-acceptance list** (non-critical ∩ not machine-verifiable): items awaiting the human's waive sign-off — recorded as accepted risk, never as verified

**In feature-workflow Step 7a** this report is the gate's input: gaps block Step 7c unless the human explicitly defers them (record the deferral in the test plan); the punt list requires the human's do-or-waive call. **Standalone**: the report is the deliverable — fixes are a separate ask.

## Remember

- Sub-clauses are the unit of verification, not cases
- Never grade an R/X clause from static reading
- Criticality is confirmed by the human before verification — never self-assigned
- Non-critical + not machine-verifiable = accepted risk with sign-off, not a model PASS
- Refute, don't confirm; uncertainty resolves down
- Batch small — late cases get first-case scrutiny
- The punt and risk-acceptance lists are human decision points, not appendices
- Date/clock cases: inject `today` (DEBUG seam) to machine-verify the *logic* through the app's real refresh path (U); the on-screen value across the change stays *R* — never let the injected-clock test speak for the pixels
