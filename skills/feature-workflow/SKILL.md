---
name: feature-workflow
description: Use when starting a new feature ("new feature", "start feature", "feature workflow") or when picking up a bug fix / mechanical refactor that should run the gated Lite Track — orchestrates the delivery lifecycle from analysis through implementation to branch completion
---

# Feature Development Workflow

**Type: Rigid** — Follow this process exactly. Do not skip steps.

Orchestrates the delivery lifecycle by invoking other skills in sequence. Each step has a human gate before proceeding.

## Invariants — apply to EVERY track, EVERY step, ALWAYS

These override anything else in this workflow. Re-read this block whenever you return to this skill.

1. **Commit policy.** Never run `git commit` / `git push` without an explicit human OK given in the *same turn*. Authorization is single-use and per-commit: a "go"/"y" from an earlier turn never carries over to a later commit. `review-task`'s `APPROVED` is the same-turn OK for *that task's* commit only — never the next task, a push, or a housekeeping commit. Subagents NEVER commit.
2. **The review gate is anchored to tasks, not documents.** Every task that changes the worktree — code, tests, assets, localization, config, or generated files; with or without a plan document, on either track — must pass `fatsecret-workflow:review-task` before its commit. Skipping planning documents never skips this gate.
3. **UI work is Figma-gated at implementation time.** Any task that creates or modifies UI invokes `fatsecret-workflow:figma-driven-implementation` with the component nodeIds before writing visual values. No nodeId → stop and ask the human for the design, or record an explicit **human-granted** design waiver in the artifacts, citing the approving turn — a waiver is never self-granted (waived UI gets no pixel-match in review).
4. **Destructive-op policy.** When removing existing callsites/files: comment out first — never hard-delete in the same pass. Enumerate every affected callsite, get the human's per-item confirmation before touching them, and run the actual deletion as a separate human-approved pass after review.
5. **Post-compaction re-anchor.** After any context compaction or session resume: re-read this Invariants block, the feature's `design.html` (the numbered D-decisions) and the execution checklist's current task where they exist — on the Lite Track, the harness task list instead — plus the `CODEBASE-KNOWLEDGE.md` task index, before doing anything else.
6. **"Proceed" / "go" / "continue with phase N" means enter that phase's first step** (analysis or design) — it is never an instruction to start editing code, unless you are already mid-task in Step 6.

## Codebase Knowledge (read first)

Before Step 2 (Analysis), Step 3 (Design), Step 5 (Execution Plan), and Step 6 (Build), read `docs/plans/CODEBASE-KNOWLEDGE.md` if it exists — the codebase map (entry points, state ownership, navigation, networking, UI/localization/analytics conventions, build/tooling diagnostics; task index at the top). Honor it during the relevant step (don't restate it in artifacts); consult it before exploring or when unsure how something is done here.

## Delegation — orchestrate, don't do everything inline

When the model running this workflow is a top-tier reasoning model and the harness supports subagents, that model's **context is the scarcest resource in the workflow** — it must survive all seven steps (compaction is the enemy; see Invariant 5). Delegate work below its pay grade; keep only decisions, gate states, and human confirmations inline. On a harness without subagents (or a lower-tier session model), run everything inline — this section is then inert; the Invariants are not.

- **Delegate DOWN: work whose spec already exists.** Read-only exploration in Steps 2–3 (the grounding rule's file verification, historical story search, hidden-impact sweeps, the grilling's "facts are looked up" legwork) — fan out read-only agents; only conclusions + `file:line` citations return to the orchestrator's context, never file dumps. In Step 6, mechanical execution of self-contained tasks (boilerplate, batch edits, tests whose assertions the plan already names). Where the harness offers per-agent model selection, use cheaper/lower-effort agents for mechanical work; when unsure, inherit the session model.
- **NEVER delegate:** anything that talks to the human (grilling, gates, `AskUserQuestion`), design decisions and their synthesis, review verdict handling, and commits (Invariant 1: subagents never commit).
- **Implementation dispatch (Step 6)** requires the full plan (existing trigger in Step 5) — and dispatches **only the task's Implement step**, never the full task text with its Review/Commit steps. The dispatch prompt must state: do not invoke `review-task`, do not ask the human, do not commit or `TaskUpdate` — implement, run the task's tests, and return a diff summary + test output. Pass *paths* to `design.html` / `CODEBASE-KNOWLEDGE.md`, don't re-summarize them into the prompt. Before invoking `review-task`, verify the files the subagent actually touched ⊆ the task's declared Files — the same discipline as the staged ⊆ Files commit rule. Review fixes may be re-dispatched, but the verdict, the human gate, and the commit stay with the orchestrator.
- **Parallel dispatch** only for tasks with disjoint Files lists, and only with each subagent in its own worktree — parallel edits in the shared tree are forbidden: `review-task`'s diff and the post-commit clean-tree check cannot isolate one task's changes from another task's uncommitted ones. After the parallel tasks have each passed review and committed, the orchestrator integrates the worktree branches back into the feature branch **serially** — each integration is itself a commit under Invariant 1 (same-turn human OK), and the combined result gets a full test run before proceeding. Integrations must preserve the reviewed commits unchanged; if one requires conflict resolution or any content change, treat it as a worktree-changing task — `review-task` before its human-authorized commit (Invariant 2).

## Prerequisites Check

Run this at Step 1 and present **one MCP readiness table** to the user before proceeding. (The sub-skills ship in this plugin — they need no availability check.) Startup detection is early warning, not a guarantee — an MCP alive now can be gone by Step 6, so the per-step preflights still own the real failure handling.

**Required MCPs** (missing → report at Step 1; the user either fixes it now or explicitly accepts failing later at the step that needs it):
- `xcodebuildmcp` — TDD in Step 6, per-task test runs in `review-task`, the 7b full-suite gate
- `figma` — only when the feature has UI work / Figma links (story-analysis, figma-driven-implementation, review-task UI verification)

**Optional MCPs** (missing → note it in the table and apply the named fallback; never block):
- `codex` — external peer reviewer. Fallback: `review-task`'s debate runs via Claude Code's built-in `/code-review` skill; `write-test-plan`'s coverage challenge runs via a fresh-context subagent. The adversarial gates themselves are NOT optional — only who plays the adversary changes.
- `proxyman` — runtime network capture in `validate-test-plan` Phase 3. Without it, R cases needing capture stay PUNTED to manual QA.
- `testsecret` — TestSecret, the authoritative case library: Step 4 case upload + Step 7a result write-back. Unreachable → defer with an explicit note to the user, never silently skip. **Not configured at all → offer to set it up** with the setup guide (https://claude.ai/code/artifact/656220b8-5cc5-46de-8880-e330839cbb62): the user creates a token in the TestSecret web UI (Settings → API tokens, shown once), then `claude mcp add testsecret --transport http <mcp-url> --header "Authorization: Bearer <token>"`; the guide also covers verification and the common failures (proxy blocks the bare IP — use the AWS hostname; hostname rotates when the instance reboots).

**Note on testing:** Tasks with testable logic are built with **TDD** — the engineer writes a failing unit test first, then implements until it passes, iterating with `xcodebuildmcp` (`build_sim` to compile, `test` to run). Test-quality rules: watch the test fail before making it pass; test observable behavior at the seams the test plan names — never internals or private methods; expected values come from an independent source of truth (a known-good literal, the spec), never recomputed the way the code computes them; one slice at a time — one test, one minimal implementation, repeat. The unit-test scope for the feature is defined in the test plan's **Unit Test Coverage** section. Per-task verification — including re-running the task's unit tests — is owned by `review-task`. UI tests and snapshot tests are out of scope; the unit test target is `Calorie Counter Tests`.

## Workflow at a Glance

```
Step 1: Introduction        — collect input, pick track, confirm output folder
  ├─ Lite Track (bug fix / mechanical refactor / small UI tweak) — see "Lite Track" below
  └─ Full Track:
     Step 2: Story Analysis      — understand stories + explore code [skip if no stories]
     Step 3: Design Exploration  — architecture decisions, grounded in real code; adversarial design challenge
     Step 4: Test Plan           — manual QA + unit-test coverage; mandatory adversarial coverage challenge
     Step 5: Execution Plan      — execution checklist (default) or full implementation plan (when triggered)
     Step 6: Build               — execute each task; review-task gates every commit
     Step 7: Verify & Ship       — blocking test-plan verification + finish branch
```

## Lite Track — bug fixes, mechanical refactors, small UI tweaks

Choose this in Step 1 when the work is a bug fix, a mechanical refactor, or a small UI change rather than a new feature — **and only when the work is genuinely small**: a refactor that is a destructive migration, spans subsystems, or needs more than ~5 tasks is Full Track work no matter how mechanical it feels. No documents are produced (no story-analysis / test-plan / plan file), but **all Invariants still apply** — most importantly the per-task review gate (Invariant 2) and the commit policy (Invariant 1).

1. **Diagnose before touching code (bug fixes).** State the suspected triggering path/layer and confirm it with evidence — a reproduction, a code trace, or logs — before editing. If you cannot confirm which layer fires the behavior, say so and keep investigating; do not fix a hypothesis. (The **Diagnosis Method** below is the method.)
2. **Minimal diff.** Fix only what the ticket describes. Never bundle changes to adjacent working code — "while I'm here" is a red flag.
3. **Task list as the execution contract.** Create harness tasks (`TaskCreate`) for the fix steps — this replaces the plan document, so each task must carry what `review-task` will review against: the exact files it will touch, its acceptance point, its unit-test scope (if there's testable logic), and the Figma nodeId or human-granted waiver for UI changes.
4. **Gated execution.** Each code task: implement (TDD if there's testable logic; `figma-driven-implementation` if UI) → `review-task` → human gate → commit per the Commit reliability rules in Step 6.
5. **Destructive changes** follow Invariant 4 (comment-out first, per-item confirmation, separate delete pass).
6. **Close out.** After the final task's commit, run Step 7b (full-suite verification); if the fix lives on its own branch, also run Step 7c (finish branch). Step 7a does not apply — there is no test plan.

### Diagnosis Method (Lite Track step 1)

Work the phases in order; skip one only when you can justify the skip aloud:

1. **Build a feedback loop first.** Before reading code to build theories, produce a repeatable pass/fail check that goes red on *this* bug and green once fixed. ONE command is the preferred form — a failing unit test at the nearest seam (`Calorie Counter Tests`), a replayed captured payload/trace, or a minimal throwaway harness kept outside the repo (scratchpad); for bugs only observable interactively (UI/device behavior), a scripted simulator run or a written manual repro sequence with a clear pass/fail criterion also qualifies. Non-deterministic bug: raise the reproduction rate (loop the trigger, pin time/RNG/state) until it's debuggable. If you genuinely cannot build a loop, stop and say so, list what you tried, and ask the human for a captured artifact or repro access — do not hypothesise without one.
2. **Reproduce, then minimise.** Confirm the loop shows the *user's exact symptom* — a nearby different failure means the wrong bug and the wrong fix. Then cut inputs, steps, and config one at a time, re-running the loop after each cut, until every remaining element is load-bearing.
3. **Hypothesise in plural.** Write 3–5 ranked, falsifiable hypotheses — "if X is the cause, changing Y makes it disappear" — before testing any; a single hypothesis anchors on the first plausible idea. Show the ranking to the human: they often re-rank it instantly.
4. **Probe one variable at a time.** A breakpoint or debugger beats ten logs; when logs are needed, tag them with a unique prefix (e.g. `[DEBUG-a4f2]`) so cleanup is one grep — never "log everything and grep". Performance bugs: measure a baseline first, then bisect; don't reach for logs.
5. **Fix at the root, prove it.** Turn the minimised repro into a failing test at a correct seam (red → fix → green), then re-run the original loop. No correct seam = a finding to surface, not a reason to fake one. Three failed fixes means the architecture is the problem — stop and discuss with the human; don't attempt a fourth.
6. **Clean up.** Grep the debug prefix and strip all instrumentation; state the confirmed root cause in the task's commit message.

## Output Directory

All workflow artifacts for a feature are saved under a single folder:

```
docs/plans/<feature-name>/
  ├── story-analysis.html       (from Step 2)
  ├── design.html               (from Step 3; includes the Execution Checklist section from Step 5)
  ├── test-plan.html            (from Step 4)
  └── implementation-plan.html  (from Step 5 — only when a full plan is triggered)
```

The feature folder name is confirmed with the user in Step 1.

### Document format — HTML with a sticky TOC

All plan artifacts are written as **self-contained HTML** (inline CSS, no external deps) with a **left-side sticky Table of Contents**. These docs are reviewed in a browser by humans (dev / QA), not in a code editor.

Use the shared skeleton at `references/plan-doc-skeleton.html` (in this skill's directory) as the starting structure for every artifact — it carries the proven CSS and the helper classes (`.item-card`, `.task-card`, `.gate`, `.callout`, `.field-label`, `.tag`/`.items-tag`, `.meta`). The sub-skills (`story-analysis`, `write-test-plan`, `writing-plans`) describe the *content* of each doc; this skeleton supplies the *rendering*.

A project may override the format in its own CLAUDE.md (e.g. `docs/plans/<area>/CLAUDE.md` "Plan document format"). If a project explicitly asks for markdown or CSV, follow that instead.

### Document language — English

**Everything this workflow writes down is in English**, regardless of the language the human and the model are conversing in: the plan artifacts above, their diagrams, the Execution Checklist, TestSecret cases, `CODEBASE-KNOWLEDGE.md` entries, code comments, and commit messages. These outlive the conversation and are read by the wider team, by QA inside TestSecret, and by tooling — the chat language is one person's channel preference, not a property of the artifact. Quoting a human's or a story's exact wording in its original language is fine where the wording itself is the point; the surrounding document stays English.

---

## Large Projects — Phase Mode

Some efforts are too large to run as a single feature (a migration, a multi-subsystem rollout). Run them in **phase mode** instead of one giant feature-workflow pass:

1. **One project-level story-analysis, done ONCE.** Run `story-analysis` for the whole project up front, saved at the project root (`docs/plans/<project>/story-analysis.html`). It covers every phase's stories and the cross-cutting decisions — you do NOT redo it per phase.
2. **Each phase is its own feature-workflow run**, scoped to a phase subfolder:
   ```
   docs/plans/<project>/
     ├── story-analysis.html            (project-wide, done ONCE)
     ├── CLAUDE.md                      (project conventions: doc format, logging, …)
     └── phase-N-<name>/
         ├── design.html                (this phase — Step 3; includes its Execution Checklist)
         ├── test-plan.html             (this phase — Step 4)
         └── implementation-plan.html   (this phase — only when a full plan is triggered)
   ```
   For each phase, treat its subfolder as the feature folder and run **Steps 3–6 plus the per-phase verification (Step 7a + 7b)**; Step 7c (finish branch) is NOT per-phase — it follows point 4's timing. **Skip Step 2's story gathering** — the project-level analysis already covers it (only re-open it if a phase surfaces genuinely new stories). But Steps 3–5 STILL take the project-root `story-analysis.html` as their story-analysis input: treat it as the *"with story analysis"* path in Step 3, not *"explore from scratch."*
3. **Design decisions are numbered and shared across phases** (`D1`, `D6`, …) so later phases reference earlier ones. Record them in each phase's `design.html` and cross-link.
4. **Settle branch strategy + Step-7 timing with the human at project start.** Phases commonly accumulate on one long-lived branch with the final finish-branch (Step 7c) run once at project end rather than per phase — but confirm this rather than assuming it; don't run a per-phase merge/PR unless the human asked for one.

When the user names a phase ("Phase 5", "P5"), that phase's subfolder is the feature folder for its Step 3–6 artifacts — the project-level `story-analysis.html` + `CLAUDE.md` stay at the project root, not in the phase folder. The rest of each step applies unchanged. A phase that is itself a destructive migration is a full-plan trigger in Step 5.

---

## Step 1 — Introduction

### 1a. Collect input & pick track

Present this to the user and collect input:

> I'll guide you through our delivery workflow:
>
> **Analyze** → understand stories, explore code for hidden impacts
> **Design** → architecture decisions grounded in the real code (adversarial design challenge)
> **Test Plan** → define acceptance criteria + unit-test coverage (adversarial coverage challenge)
> **Plan** → execution checklist (or a full plan for complex work)
> **Build** → implement (TDD unit tests), review each task, commit
>
> To get started — do you have **Shortcut Stories** (Story IDs or Iteration ID) or **Figma designs** to share?

Collect Story IDs, Iteration IDs, and Figma URLs as provided.

**Track check:** if the work is a bug fix, a mechanical refactor, or a small UI tweak rather than a new feature, offer the **Lite Track** (see above); if chosen, run the Lite Track's steps **instead of Steps 2–7** (its close-out note covers verification and branch finish). When in doubt, ask.

### 1b. Scale check — offer phase mode

Before naming the output folder, judge the SIZE of what was collected. Offer **phase mode** (see "Large Projects — Phase Mode" above) ONLY when the work is a large epic / iteration that obviously spans multiple weeks and multiple releases — concretely, when **either** holds:

- the stories span **more than one iteration**, or
- there are **10+ stories**.

If neither holds, say nothing — proceed as a single feature run. Don't offer phase mode for a handful of stories in one iteration; it's overhead a small feature doesn't need.

When the threshold is met, use `AskUserQuestion` to offer the choice:
- **Single feature run** — one pass of Steps 2–7, all artifacts in one folder.
- **Phase mode** — one project-level story-analysis up front, then each phase run as its own pass (Steps 3–6 + 7a/7b).

If the user picks phase mode, record the decision + the phase breakdown in the project-root `CLAUDE.md` so every later phase run inherits it without re-asking. If the project is already set up in phases (a project-root `story-analysis.html` + `phase-*/` folders already exist), phase mode is already in effect — skip the question and proceed with the named phase.

### 1c. Confirm output folder

> "I'll save all artifacts to `docs/plans/<feature-name>/` — does this name work?"

(Phase mode: this phase's artifacts go in `docs/plans/<project>/phase-N-<name>/`; the project-level `story-analysis.html` + `CLAUDE.md` stay at the project root — confirm both.)

---

## Step 2 — Story Analysis [human gate]

**Skill**: `fatsecret-workflow:story-analysis`
**Condition**: Stories were provided (Story IDs, Iteration ID, or pasted text). If no stories, skip to Step 3.

Save output to `docs/plans/<feature-name>/story-analysis.html`.

---

## Step 3 — Design Exploration [human gate]

**Condition**: Always (Full Track). Design is done inline — do not invoke other workflow sub-skills for this step (the `archify` diagram skill below is the one exception).

**Input varies by path:**
- **With story analysis** (Step 2 completed): Uses story-analysis output as input. Objectives: (1) Propose implementation approaches and architecture decisions for the Items (2) Identify existing code issues that affect this work (3) Design shared components and state management across Items
- **Without story analysis** (Step 2 skipped): Explore the idea from scratch — understand intent, propose approaches, present design.

**Design grilling — interview before proposing.** On both paths, design starts as a relentless interview, not a proposal:

1. **Load the docs first.** The feature's stories are the spec: re-read the story-analysis output (or the user's raw request when there are no stories), and search Shortcut for related *historical* stories — same epic, module, or feature area — as background: what was shipped, decided, or explicitly rejected before. Cite story IDs when a past decision constrains this design. If Shortcut MCP is unavailable, record "historical search unavailable" in `design.html` and ask the human for relevant past stories instead of silently skipping.
2. **Facts are looked up, decisions are asked.** Anything answerable from the codebase, the stories, or `docs/plans/CODEBASE-KNOWLEDGE.md` — look it up yourself; never ask the human for it. Genuine decisions (scope, trade-off preferences, UX intent) go to the human **one question at a time, each with your recommended answer**, walking the decision tree branch by branch so dependent decisions are resolved in order.
3. **Sharpen the language.** Challenge vague or overloaded terms ("you said 'account' — the Customer or the User?"). When the human's claim contradicts the code or a historical story, surface the contradiction with the evidence. Resolved terminology is recorded in `design.html`; a durable codebase-wide fact is proposed for `CODEBASE-KNOWLEDGE.md` (human yes required, as in the Knowledge Delta Check).
4. **Stress-test with scenarios.** Invent concrete edge-case scenarios that force precision about the boundaries between concepts before any code is designed around them.
5. **Only then propose 2–3 approaches** with trade-offs and a recommendation. Decisions that are hard to reverse and came from a real trade-off get their rationale recorded against their D-number in `design.html` — a future reader should never wonder "why did they do it this way?".

Do not present a final design until the interview has reached shared understanding — concretely, until every exit criterion holds:

- **Every acceptance point names its verification mechanism** — a unit-test assertion, a Figma nodeId, a manual QA step for the test plan, or a measurable threshold. A criterion nothing can verify is renegotiated now, not discovered at Step 7a.
- **Behaviors carry concrete input → expected-output examples** (edge and error cases included) from the scenario stress-tests, ready to become test-plan assertions — the human's examples are the independent source of truth TDD asserts against.
- **A non-goals / out-of-scope list is recorded in `design.html`** — prohibitions are executable; "keep it minimal" is not.
- **Known goal conflicts have a pre-ranked priority** ("when X conflicts with Y, prefer X"), captured with the relevant D-decision.
- **Qualitative goals are converted** to a measurable threshold or explicitly marked "no threshold — human judges in QA"; never left as adjectives.
- **Everything decided in the interview is written into `design.html`.** A decision that lives only in the conversation was not decided — compaction erases it, and Invariant 5 re-anchors on the artifact, not the chat.

Explore the codebase, discuss trade-offs, and write up the design. Before it goes to the human it must survive the **Design Challenge** below; only the folded-in version is presented. Wait for human confirmation.

**Grounding rule — verify, don't remember.** Every type, file, field, wire format, or claim about existing behavior that appears in the design MUST be verified against the actual codebase before it is presented — open the code and cite the source path in `design.html`. This is the `figma-driven-implementation` discipline applied to code facts: query, don't remember. A design that names a type or API you haven't opened this session is not ready to present. When replacing an existing subsystem, the source implementation is the spec — read and enumerate its guards/branches/semantics before designing the replacement (see `writing-plans` "Migration discipline").

**Assumptions & Risks — mandatory section in `design.html`.** Any load-bearing claim you could NOT verify from code (third-party SDK behavior, backend contract, upgrade-state semantics) goes here: the assumption + how it will be verified (read SDK source / ask human / spike task). Do not pad the list — if everything was verified, write "None". An unverified assumption the design depends on is a risk, not a footnote.

Save output to `docs/plans/<feature-name>/design.html`.

**Data-flow / flow diagram (draw it when the shape carries information).** Include a diagram in `design.html` whenever the phase's shape says something prose can't show at a glance — and skip it when prose/tables already do. Draw it for: a chain across several components (entry → processing → local store → network → read-back); a **fan-in / fan-out or source-of-truth merge** (a data migration that unions multiple sources, or a dual-write that splits into several); a local↔network (or sync↔async) boundary crossing; guarded branches where a condition changes the path; or a state machine / transition. **Skip** it for a single function or component, a static data model or contract (use a table), pure-UI layout (Figma owns that), and trivial linear two-step calls (a sentence suffices). Show the *main flow*: the classes and key functions it passes through, and how data is processed, persisted, uploaded, and downloaded. Author it **at design time — before the build** (not as an after-the-fact "as-built" diagram): it is the architecture spine the execution plan slices into tasks; after build, annotate only the deltas.

Render it with the **`archify` skill** — do not hand-author SVG or use real Mermaid/CDN. Invoke `archify` and pick the mode that matches the shape: `dataflow` (pipelines, migration/seeding, source-of-truth merge, cross-device sync, lineage) · `workflow` (flows, guarded branches, approval gates, runbooks) · `sequence` (call chains / request lifecycles — who calls whom) · `lifecycle` (state machines, transitions, retries, terminal states) · `architecture` (component / system maps). archify emits a **self-contained, themed HTML diagram** (dark/light toggle + PNG/JPEG/WebP/SVG export) with validated layout, and also accepts pasted Mermaid (it re-lays-it-out from scratch). Save the diagram into `docs/plans/<feature-name>/` beside `design.html` and link it from there. Skip it for pure-UI or trivial single-component phases. (If the `archify` skill isn't installed, fall back to a self-contained inline SVG following the same when-to-draw rules.)

### Design Challenge (devil's advocate) — mandatory

**Type: Rigid gate. Do not skip, do not rationalize past it.** Once `design.html` is written and **before it reaches the human**, the design MUST survive an adversarial review — by Codex when available, by a fresh-context subagent otherwise. The human gate reviews the *challenged* design, never the first draft.

1. Open a Codex session with `mcp__codex__codex`, `cwd` = project root, `sandbox: read-only`, `approval-policy: never` — the challenger needs the real code to check the design against.
2. Send `design.html` (plus the story-analysis output, or the raw request when there are no stories) and frame Codex as a **picky devil's advocate** whose only job is to attack the design. Ask it to find:
   - **Ungrounded claims** — every type, file, field, wire format, or existing-behavior claim in the design, checked against the actual code. This is the highest-value find: the author cannot reliably self-check what they believe they verified.
   - **Wrong decisions** — a rejected alternative that is actually better, a D-decision whose stated trade-off doesn't exist, a rationale that is post-hoc justification
   - **Missing Assumptions & Risks** — load-bearing claims presented as fact that were never verified against anything
   - **Unverifiable acceptance** — acceptance points naming no verification mechanism, qualitative goals left as adjectives, absent non-goals, goal conflicts with no pre-ranked priority
   - **Uncovered paths** — error/failure handling, concurrency, migration/rollback, and regressions of existing behavior the design never mentions
3. Evaluate each challenge with project context — accept real findings, push back (with reasoning) on speculative ones. **Push back hardest on scope creep**: an adversary told to attack a design reflexively proposes more architecture. The bar is "this design is *wrong or incomplete*", never "this design could be grander" — simplicity survives unless the challenger names a concrete failure it causes. Iterate via `mcp__codex__codex-reply`; keep the thread.
4. Fold the agreed findings into `design.html`. Only then present the design to the human, and say what the challenge changed.

**Codex unavailable → internal fallback, not a stop**: hand the same brief to a **fresh-context subagent** (`general-purpose`, read-only) that took no part in producing the design. Evaluate and fold its findings exactly as in steps 3–4, and note in the summary that the challenge ran on the internal fallback. The gate is on the adversarial pass, not on Codex specifically — it is never skipped.

| Thought | Reality |
|---|---|
| "The grilling interview already stress-tested this" | The interview tested the human's intent. The challenge tests the *design against the code* — a different axis. |
| "I verified every claim as I wrote it" | Self-verification is exactly what this catches; you cannot audit your own recall. |
| "The design is small / obvious" | Then the challenge costs one round and returns nothing. Run it. |
| "Auto mode means I can skip this" | Auto mode does not disable a Rigid gate. |

---

## Step 4 — Test Plan [human gate]

**Skill**: `fatsecret-workflow:write-test-plan`
**Condition**: Always (Full Track).

Uses the story-analysis output (if available) and design exploration output as input.

The test plan defines both the **manual QA acceptance cases** and a separate **Unit Test Coverage** section (the deterministic, non-UI units that get TDD unit tests in Step 6; target `Calorie Counter Tests`, no UI/snapshot tests). Before the plan is finalized, `write-test-plan` runs a **mandatory devil's-advocate coverage challenge** over the whole draft (manual + unit) — Codex when available, a fresh-context subagent otherwise; the gate itself is not skippable.

Save output to `docs/plans/<feature-name>/test-plan.html`.

Wait for human confirmation before proceeding. **After confirmation**, `write-test-plan` uploads the approved manual QA cases to TestSecret (`testsecret` MCP) as `Draft` — section and tag named after the feature, TestSecret case ids written back into `test-plan.html`. TestSecret is the authoritative case library from that point on; Step 7a validates against it and records results there.

---

## Step 5 — Execution Plan [human gate]

**Skill**: `fatsecret-workflow:writing-plans`
**Condition**: Always (Full Track).

Uses all prior outputs (story-analysis, design, test plan) as input.

**Default: an Execution Checklist, not a separate document.** For most features, `writing-plans` produces a lightweight checklist appended to `design.html` — one entry per task: title, exact Files (+ Figma nodeId for UI files), the unit tests it owns (from the test plan's Unit Test Coverage), its acceptance point, and the standard Implement → Review → Commit steps. **No code blocks** — implementers read the real files at task time.

**Write a full `implementation-plan.html` only when any trigger holds:** destructive migration; multiple subsystems; tasks will be dispatched to subagents; the user asks for one.

**Folder placement for new files** is confirmed with the human ONCE, as a single batched `AskUserQuestion` (each new file: proposed folder + one-line rationale) — not per file.

**Figma coverage:** every UI task must carry its Figma nodeId in the checklist/plan. If a UI element has no design node, ask the user for the design — or record an explicit waiver — before the checklist is finalized. (Implementation-time enforcement is Invariant 3, so this holds even when this step's documents are skipped.)

Wait for human confirmation before proceeding.

---

## Step 6 — Build

**Condition**: Always.

### Pre-task check

Before starting each new task, run `git status` to confirm the prior task committed cleanly. If uncommitted changes remain, investigate the root cause before starting the new task.

**Task brief — announce before touching anything.** Open every task by presenting a short, visually prominent brief (bold header + bullets) restating its checklist/plan entry — never invented, never expanded:

> **▶ Task N — \<title\>**
> - **Goal**: the observable outcome this task delivers — its acceptance point
> - **Files**: the declared Files list (+ Figma nodeId for UI files)
> - **Tests**: the unit tests this task owns, or "none — pure UI"

This is the checklist entry re-read aloud, not a re-plan: it re-anchors on the artifact at every task boundary (the same discipline Invariant 5 applies after compaction) and gives the human a glanceable checkpoint — a surprising brief gets caught before code is written, not at review. If what you were about to do doesn't match the brief, stop and re-read the checklist instead of improvising. Delegated tasks carry the same brief at the top of the dispatch prompt.

### Execution

Execute each task exactly as written in the execution checklist (or full plan). **The checklist/plan is the single source of truth for the steps inside a task.**

Invariants Step 6 enforces on top of the checklist/plan:

- `review-task` performs ONE review pass per invocation and returns its report. **The caller (you) owns the review-fix loop**: fix the reported issues (fixes only — no commit), re-invoke `review-task`, at most 3 rounds; after 3 rounds without an APPROVE recommendation, treat the task as `ESCALATED`.
- `review-task`'s `APPROVED` verdict unlocks the task's commit + `TaskUpdate(completed)`. On `ESCALATED`, halt the task and surface the issues to the human.
- After `APPROVED` and before the commit, run the **Knowledge Delta Check** (defined in `writing-plans`): only if the review surfaced a human correction about how this codebase works, or a conflict with `docs/plans/CODEBASE-KNOWLEDGE.md`, propose the addition/fix to the human (write only on an explicit yes); otherwise skip silently. Never blocks the commit.
- **Commit reliability:** stage exactly the task's declared Files (`git add <paths>` — never `git add -A` / `git add .`); before committing, verify staged paths ⊆ the task's Files list and question anything extra; after committing, run `git status` and confirm the tree is clean before declaring the task complete.

### Design change mid-build (re-entry procedure)

When the design changes after tasks have been built (a new meeting decision, a revised backend contract):

1. Update `design.html` first — new/changed D-numbers — and diff the old vs new decisions.
2. List every already-completed task the delta affects.
3. Confirm the list with the human, then fix the affected completed tasks FIRST (each fix passes the normal review → commit gate) before building new tasks.

---

## Step 7 — Verify & Ship

Three sub-steps to close out the feature:

### 7a. Test plan verification [blocking]

**Skill**: `fatsecret-workflow:validate-test-plan`
**Condition**: Always (Full Track).

Validate implementation coverage against the test plan — refute-first, typed verdicts per sub-clause, runtime/external clauses punted explicitly rather than graded from static reading. **Blocking:** unresolved acceptance gaps stop the workflow here — do not proceed to 7c unless the user explicitly defers a gap (record the deferral and reason in the test plan); each punt-list item needs the human's do-or-waive call. For features whose per-task review gates were structurally weak — no Figma AND no unit-testable logic (e.g. backend-integration work) — 7a must include verification against the real backend/integration (the skill's runtime pass), or an explicit user waiver.

After the human resolves the report, the skill records the outcome to TestSecret (Phase 5): a run whose per-case results carry the verdicts (`Dev-Verified` / `AI-Tested` / `Failed` / `Retest` / `Skipped`), with punted cases left `Untested` — that Untested set is QA's manual queue.

### 7b. Verify before completion

**Condition**: Always.

**Evidence before claims.** Run the full unit-test suite fresh (`xcodebuildmcp` `test`, target `Calorie Counter Tests`) and read the output — the per-task gates already ran each task's tests; this confirms the whole suite is green before shipping. Any completion claim ("tests pass", "build succeeds", "bug fixed") is made only after its verification command has run *in this session* with output confirming it — no "should pass", no extrapolating from an earlier run, no trusting a subagent's success report without reading the diff. If the target can't be built/run (e.g., a non-app worktree), surface that rather than silently skipping.

### 7c. Finish branch

**Condition**: Always.

With 7b green, determine the base branch (the branch this work forked from — typically `main`; confirm with the human if in doubt), then present exactly these options and execute the choice (merge/push still follow Invariant 1 — the human's same-turn choice is the OK):

> Implementation complete. What would you like to do?
> 1. Merge back to `<base-branch>` locally
> 2. Push and create a Pull Request
> 3. Keep the branch as-is (I'll handle it later)
> 4. Discard this work

- **1 — Merge:** checkout base → `git pull --ff-only` (a pull that would create a merge commit stops here — surface it and ask) → merge → re-run the test suite on the merged result → only then clean up: worktree first (rules below), then `git branch -d`.
- **2 — PR:** `git push -u origin <branch>` and open the PR. Keep the worktree — it's needed to iterate on PR feedback.
- **3 — Keep:** report the branch name and worktree path; touch nothing.
- **4 — Discard:** list the branch, its commits, and the worktree that will be deleted; require the human to type `discard`; only then run worktree cleanup + `git branch -D`.
- **Worktree cleanup (options 1 and 4 only):** only remove worktrees under `.worktrees/` or `worktrees/` — ours by provenance. `cd` to the main repo root first, `git worktree remove <path>` then `git worktree prune`, always before deleting the branch. Harness-owned workspaces are left in place (use the harness's exit tool if one exists).

---

## Important

- Do NOT skip steps. If a conditional step does not apply, move to the next.
- Every step with a human gate requires explicit user confirmation before proceeding.
- All artifacts save to the feature's output directory, not the skill's default location.
- The **Invariants** block at the top applies to every track and every step — when in doubt, re-read it.
