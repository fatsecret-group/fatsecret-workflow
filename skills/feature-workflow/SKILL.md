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
3. **UI work is Figma-gated at implementation time.** Any task that creates or modifies UI invokes `fatsecret-workflow:figma-driven-implementation` with the component nodeIds before writing visual values. No nodeId → stop and ask the human for the design, or record an explicit **human-granted** design waiver in the artifacts, citing the approving turn — a waiver is never self-granted (waived UI gets no pixel-match in review). A waiver covers visual styling only: placement, ordering and visibility decisions (where an entry point sits, what is shown or hidden) are never guessed under it — they go to design or PM.
4. **Destructive-op policy.** When removing existing callsites/files: comment out first — never hard-delete in the same pass. Enumerate every affected callsite, get the human's per-item confirmation before touching them, and run the actual deletion as a separate human-approved pass after review.
5. **Post-compaction re-anchor.** After any context compaction or session resume: re-read this Invariants block, the feature's `design.html` (the numbered D-decisions) and the execution checklist's current task where they exist — on the Lite Track, the harness task list instead — plus the `CODEBASE-KNOWLEDGE.md` task index, before doing anything else.
6. **"Proceed" / "go" / "continue with phase N" means enter that phase's first step** (analysis or design) — it is never an instruction to start editing code, unless you are already mid-task in Step 6.

## Codebase Knowledge (read first)

Before Step 2 (Analysis), Step 3 (Design), Step 5 (Execution Plan), and Step 6 (Build), read `docs/plans/CODEBASE-KNOWLEDGE.md` if it exists — the codebase map (entry points, state ownership, navigation, networking, UI/localization/analytics conventions, build/tooling diagnostics; task index at the top). Honor it during the relevant step (don't restate it in artifacts); consult it before exploring or when unsure how something is done here.

## Delegation — orchestrate, don't do everything inline

When the model running this workflow is a top-tier reasoning model and the harness supports subagents, that model's **context is the scarcest resource in the workflow** — it must survive all seven steps (compaction is the enemy; see Invariant 5). Delegate work below its pay grade; keep only decisions, gate states, and human confirmations inline. On a harness without subagents (or a lower-tier session model), run everything inline — this section is then inert; the Invariants are not.

- **Delegate DOWN: work whose spec already exists.** Read-only exploration in Steps 2–3 (the grounding rule's file verification, historical story search, hidden-impact sweeps, the grilling's "facts are looked up" legwork) — fan out read-only agents; only conclusions + `file:line` citations return to the orchestrator's context, never file dumps. In Step 6, mechanical execution of self-contained tasks (boilerplate, batch edits, tests whose assertions the plan already names). Where the harness offers per-agent model selection, use cheaper/lower-effort agents for mechanical work; when unsure, inherit the session model.
- **NEVER delegate:** anything that talks to the human (grilling, gates, `AskUserQuestion`), design decisions and their synthesis, review verdict handling, and commits (Invariant 1: subagents never commit).
- **Implementation dispatch (Step 6)** requires the task's checklist entry to be self-contained (`writing-plans` "One Artifact": interface signatures + the concrete verification for each of its rows) — and dispatches **only the task's Implement step**, never the full task text with its Review/Commit steps. The dispatch prompt must state: do not invoke `review-task`, do not ask the human, do not commit or `TaskUpdate` — implement, run the task's verification, and return a diff summary + its output. Pass *paths* to `design.html` / `CODEBASE-KNOWLEDGE.md`, don't re-summarize them into the prompt. Before invoking `review-task`, verify the files the subagent actually touched ⊆ the task's declared Files — the same discipline as the staged ⊆ Files commit rule. Review fixes may be re-dispatched, but the verdict, the human gate, and the commit stay with the orchestrator.
- **Parallel dispatch** only for tasks with disjoint Files lists, and only with each subagent in its own worktree — parallel edits in the shared tree are forbidden: `review-task`'s diff and the post-commit clean-tree check cannot isolate one task's changes from another task's uncommitted ones. After the parallel tasks have each passed review and committed, the orchestrator integrates the worktree branches back into the feature branch **serially** — each integration is itself a commit under Invariant 1 (same-turn human OK), and the integrated tasks' declared verification (their Verification rows + Re-run lists) is re-run on the combined result before proceeding. Integrations must preserve the reviewed commits unchanged; if one requires conflict resolution or any content change, treat it as a worktree-changing task — `review-task` before its human-authorized commit (Invariant 2).

## Prerequisites Check

Run this at Step 1 and present **one readiness table** (MCP servers + the Codex CLI) to the user before proceeding. (The sub-skills ship in this plugin — they need no availability check.) Startup detection is early warning, not a guarantee — an MCP alive now can be gone by Step 6, so the per-step preflights still own the real failure handling.

**Required MCPs** (missing → report at Step 1; the user either fixes it now or explicitly accepts failing later at the step that needs it):
- `xcodebuildmcp` — builds, scoped test runs and the simulator pass in Step 6 / `review-task`
- `figma` — only when the feature has UI work / Figma links (story-analysis, figma-driven-implementation, review-task UI verification)

**Optional tools** (missing → note it in the table and apply the named fallback; never block):
- Codex CLI (`codex`) — external peer reviewer for the three adversarial gates (design challenge, coverage challenge, `review-task` debate). Called through the **Codex CLI** (`codex exec --sandbox read-only`; `codex review` for a diff) — Codex has no MCP server. CLI not installed or not signed in → the design and coverage challenges run via a fresh-context subagent and `review-task`'s debate via Claude Code's built-in `/code-review`. The adversarial gates themselves are NOT optional — only who plays the adversary changes.
- `proxyman` — network capture for script-row verification (Step 6 / `review-task`) and `validate-test-plan` Phase 3. Without it, script rows assert on the script's own input/output — except rows about a request the app itself sends, which only a capture can observe: those are reported UNVERIFIED and carried to Step 7a — and capture-dependent R cases stay PUNTED to manual QA.
- `shortcut` — reads the stories (Step 2) and posts the decision summary to them (Step 3 gate). Unavailable → the human pastes the stories in, and the approved summary is handed to the human to paste onto each story.
- `testsecret` — TestSecret, where QA keeps their test cases: Step 4 uploads the draft cases as Draft; Step 7a validates the feature's cases and writes results back. Unreachable or not configured → the upload is deferred with a note in `test-plan.html` and Step 7a validates the dev side only, both stated to the user (setup guide: https://claude.ai/code/artifact/656220b8-5cc5-46de-8880-e330839cbb62).

**Worktree readiness** (same table): a fresh git worktree does not carry git-excluded local files. Before Step 2, confirm the working copy has the project's `REVIEW.md` (when the project keeps one), the dependencies the build needs (e.g. `Pods/` in sync with its lockfile), and the project-scoped MCP config (`.mcp.json`) — copy or link them from the main checkout when missing.

**Note on verification:** every deliverable is verified by the method its **Verification Plan** row names — the methods and their rules are defined once, in `write-test-plan` (Step 4). Two rules hold everywhere, the Lite Track included: **unit tests are written ONLY for wrong-answer-is-a-bug logic** (calculations, conversions, percentage/date arithmetic, state transitions) — failing-first, asserting observable behavior against an independent source of truth; and **test runs are always scoped** (`extraArgs: ["-only-testing:Calorie Counter Tests/<TestClass>"]`), never the unfiltered target. Per-task re-verification is owned by `review-task`. UI tests and snapshot tests are out of scope.

## Workflow at a Glance

```
Step 1: Introduction        — collect input, pick track, confirm output folder
  ├─ Lite Track (bug fix / mechanical refactor / small UI tweak) — see "Lite Track" below
  └─ Full Track:
     Step 2: Story Analysis      — understand stories + explore code [skip if no stories]
     Step 3: Design Exploration  — architecture decisions, grounded in real code; adversarial design challenge; implementation Q&A; decisions posted to the Shortcut stories for QA
     Step 4: Test Plan           — dev's Verification Plan (in design.html) + draft QA cases (test-plan.html, uploaded to TestSecret as Draft); mandatory adversarial coverage challenge
     Step 5: Execution Plan      — execution checklist appended to design.html
     Step 6: Build               — execute each task; review-task gates every commit
     Step 7: Verify & Ship       — blocking deliverable verification (+ the feature's TestSecret cases) + finish branch
```

## Lite Track — bug fixes, mechanical refactors, small UI tweaks

Choose this in Step 1 when the work is a bug fix, a mechanical refactor, or a small UI change rather than a new feature — **and only when the work is genuinely small**: a refactor that is a destructive migration, spans subsystems, or needs more than ~5 tasks is Full Track work no matter how mechanical it feels. No documents are produced (no story-analysis, design, Verification Plan, test plan or checklist), but **all Invariants still apply** — most importantly the per-task review gate (Invariant 2) and the commit policy (Invariant 1).

1. **Diagnose before touching code (bug fixes).** State the suspected triggering path/layer and confirm it with evidence — a reproduction, a code trace, or logs — before editing. If you cannot confirm which layer fires the behavior, say so and keep investigating; do not fix a hypothesis. (The **Diagnosis Method** below is the method.)
2. **Minimal diff.** Fix only what the ticket describes. Never bundle changes to adjacent working code — "while I'm here" is a red flag.
3. **Task list as the execution contract.** Create harness tasks (`TaskCreate`) for the fix steps — this replaces the plan document, so each task must carry what `review-task` will review against: the exact files it will touch, its acceptance point, how it will be verified (a failing-first unit test only for wrong-answer-is-a-bug logic; otherwise the check that proves the fix), and the Figma nodeId or human-granted waiver for UI changes.
4. **Gated execution.** Each code task: implement (failing-first unit test only if the fix is wrong-answer-is-a-bug logic; `figma-driven-implementation` if UI) → `review-task` → human gate → commit per the Commit reliability rules in Step 6.
5. **Destructive changes** follow Invariant 4 (comment-out first, per-item confirmation, separate delete pass).
6. **Close out.** After the final task's commit, if the fix lives on its own branch, run Step 7b (finish branch). Step 7a does not apply — there is no Verification Plan.

### Diagnosis Method (Lite Track step 1)

Work the phases in order; skip one only when you can justify the skip aloud:

1. **Build a feedback loop first.** Before reading code to build theories, produce a repeatable pass/fail check that goes red on *this* bug and green once fixed. ONE command is the preferred form — a replayed captured payload/trace, a minimal throwaway harness kept outside the repo (scratchpad), or, for wrong-answer logic, a failing unit test at the nearest seam (`Calorie Counter Tests`); for bugs only observable interactively (UI/device behavior), a scripted simulator run or a written manual repro sequence with a clear pass/fail criterion also qualifies. Non-deterministic bug: raise the reproduction rate (loop the trigger, pin time/RNG/state) until it's debuggable. If you genuinely cannot build a loop, stop and say so, list what you tried, and ask the human for a captured artifact or repro access — do not hypothesise without one.
2. **Reproduce, then minimise.** Confirm the loop shows the *user's exact symptom* — a nearby different failure means the wrong bug and the wrong fix. Then cut inputs, steps, and config one at a time, re-running the loop after each cut, until every remaining element is load-bearing.
3. **Hypothesise in plural.** Write 3–5 ranked, falsifiable hypotheses — "if X is the cause, changing Y makes it disappear" — before testing any; a single hypothesis anchors on the first plausible idea. Show the ranking to the human: they often re-rank it instantly.
4. **Probe one variable at a time.** A breakpoint or debugger beats ten logs; when logs are needed, tag them with a unique prefix (e.g. `[DEBUG-a4f2]`) so cleanup is one grep — never "log everything and grep". Performance bugs: measure a baseline first, then bisect; don't reach for logs.
5. **Fix at the root, prove it.** Fix the root cause, then re-run the original loop and watch it go green — that run is the proof. Keep a regression test only when the risk warrants one: wrong-answer-is-a-bug logic, money / entitlements / persisted data, or a bug that has come back before. A short, obvious fix needs no test. Three failed fixes means the architecture is the problem — stop and discuss with the human; don't attempt a fourth.
6. **Clean up.** Grep the debug prefix and strip all instrumentation; state the confirmed root cause in the task's commit message.

## Output Directory

All workflow artifacts for a feature are saved under a single folder:

```
docs/plans/<feature-name>/
  ├── story-analysis.html       (from Step 2)
  ├── design.html               (from Step 3; includes the Verification Plan from Step 4 and the Execution Checklist from Step 5)
  └── test-plan.html            (from Step 4 — draft QA cases, uploaded to TestSecret)
```

The feature folder name is confirmed with the user in Step 1. `docs/plans/` and `verification/` are relative to the project's **source root** — the folder that already holds `docs/plans/` (in some repos an app subfolder, not the git root).

### Document format — HTML with a sticky TOC

All plan artifacts are written as **self-contained HTML** (inline CSS, no external deps) with a **left-side sticky Table of Contents**. These docs are reviewed in a browser by humans (dev / QA), not in a code editor.

Use the shared skeleton at `references/plan-doc-skeleton.html` (in this skill's directory) as the starting structure for every artifact — it carries the proven CSS and the helper classes (`.item-card`, `.task-card`, `.gate`, `.callout`, `.field-label`, `.tag`/`.items-tag`, `.meta`). The sub-skills (`story-analysis`, `write-test-plan`, `writing-plans`) describe the *content* of each doc; this skeleton supplies the *rendering*.

A project may override the format in its own CLAUDE.md (e.g. `docs/plans/<area>/CLAUDE.md` "Plan document format"). If a project explicitly asks for markdown, follow that instead.

### Document language — English

**Everything this workflow writes down is in English**, regardless of the language the human and the model are conversing in: the plan artifacts above, their diagrams, the Verification Plan and Execution Checklist, the decision summaries posted to Shortcut stories, TestSecret cases and result comments, `CODEBASE-KNOWLEDGE.md` entries, code comments, and commit messages. These outlive the conversation and are read by the wider team, by QA on the stories and in TestSecret, and by tooling — the chat language is one person's channel preference, not a property of the artifact. Quoting a human's or a story's exact wording in its original language is fine where the wording itself is the point; the surrounding document stays English.

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
         ├── design.html                (this phase — Step 3; includes its Verification Plan and Execution Checklist)
         └── test-plan.html             (this phase — Step 4)
   ```
   For each phase, treat its subfolder as the feature folder and run **Steps 3–6 plus the per-phase verification (Step 7a)**; Step 7b (finish branch) is NOT per-phase — it follows point 4's timing. **Skip Step 2's story gathering** — the project-level analysis already covers it (only re-open it if a phase surfaces genuinely new stories). But Steps 3–5 STILL take the project-root `story-analysis.html` as their story-analysis input: treat it as the *"with story analysis"* path in Step 3, not *"explore from scratch."*
3. **Design decisions are numbered and shared across phases** (`D1`, `D6`, …) so later phases reference earlier ones. Record them in each phase's `design.html` and cross-link.
4. **Settle branch strategy + Step-7 timing with the human at project start.** Phases commonly accumulate on one long-lived branch with the final finish-branch (Step 7b) run once at project end rather than per phase — but confirm this rather than assuming it; don't run a per-phase merge/PR unless the human asked for one.

When the user names a phase ("Phase 5", "P5"), that phase's subfolder is the feature folder for its Step 3–6 artifacts — the project-level `story-analysis.html` + `CLAUDE.md` stay at the project root, not in the phase folder. The rest of each step applies unchanged. A phase that is itself a destructive migration follows `writing-plans` "Migration Discipline" in Step 5.

---

## Step 1 — Introduction

### 1a. Collect input & pick track

Present this to the user and collect input:

> I'll guide you through our delivery workflow:
>
> **Analyze** → understand stories, explore code for hidden impacts
> **Design** → architecture decisions grounded in the real code (adversarial design challenge, then implementation Q&A)
> **Share with QA** → post the agreed decisions to the Shortcut stories, so QA writes their cases from the same understanding
> **Test Plan** → how dev proves each deliverable done, plus draft QA cases uploaded to TestSecret (adversarial coverage challenge)
> **Plan** → execution checklist
> **Build** → implement, verify each task per its deliverable criteria, review, commit
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
- **Phase mode** — one project-level story-analysis up front, then each phase run as its own pass (Steps 3–6 + 7a).

If the user picks phase mode, record the decision + the phase breakdown in the project-root `CLAUDE.md` so every later phase run inherits it without re-asking. If the project is already set up in phases (a project-root `story-analysis.html` + `phase-*/` folders already exist), phase mode is already in effect — skip the question and proceed with the named phase.

### 1c. Confirm output folder

> "I'll save all artifacts to `docs/plans/<feature-name>/` — does this name work?"

State the resolved absolute path and the branch (and worktree, when in one) the artifacts will be written on.

(Phase mode: this phase's artifacts go in `docs/plans/<project>/phase-N-<name>/`; the project-level `story-analysis.html` + `CLAUDE.md` stay at the project root — confirm both.)

---

## Step 2 — Story Analysis [human gate]

**Skill**: `fatsecret-workflow:story-analysis`
**Condition**: Stories were provided (Story IDs, Iteration ID, or pasted text). If no stories, skip to Step 3.

Save output to `docs/plans/<feature-name>/story-analysis.html`.

---

## Step 3 — Design Exploration [human gate]

**Condition**: Always (Full Track). Design is done inline — do not invoke other workflow sub-skills for this step.

**Input varies by path:**
- **With story analysis** (Step 2 completed): Uses story-analysis output as input. Objectives: (1) Propose implementation approaches and architecture decisions for the Items (2) Identify existing code issues that affect this work (3) Design shared components and state management across Items
- **Without story analysis** (Step 2 skipped): Explore the idea from scratch — understand intent, propose approaches, present design.

**Design grilling — interview before proposing.** On both paths, design starts as a relentless interview, not a proposal:

1. **Load the docs first.** The feature's stories are the spec: re-read the story-analysis output (or the user's raw request when there are no stories), and bring in the *historical* stories as background — what was shipped, decided, or explicitly rejected before. `story-analysis` already ran this search when Step 2 ran; search Shortcut yourself (same epic, module, or feature area) only when Step 2 was skipped or skipped its search. Cite story IDs when a past decision constrains this design. If Shortcut MCP is unavailable, record "historical search unavailable" in `design.html` and ask the human for relevant past stories instead of silently skipping.
2. **Facts are looked up, decisions are asked.** Anything answerable from the codebase, the stories, or `docs/plans/CODEBASE-KNOWLEDGE.md` — look it up yourself; never ask the human for it. Genuine decisions (scope, trade-off preferences, UX intent) go to the human **one question at a time, each with your recommended answer** — except placement / ordering / visibility questions under a design waiver, where the recommendation is "ask design or PM", never a guess — walking the decision tree branch by branch so dependent decisions are resolved in order. A decision the human has already recorded — a story comment, an earlier `design.html`, a decision file they point to — is cited with its source, not re-asked; only an uncertain match goes back to them.
3. **Sharpen the language.** Challenge vague or overloaded terms ("you said 'account' — the Customer or the User?"). When the human's claim contradicts the code or a historical story, surface the contradiction with the evidence. Resolved terminology is recorded in `design.html`; a durable codebase-wide fact is proposed for `CODEBASE-KNOWLEDGE.md` (human yes required, as in the Knowledge Delta Check).
4. **Stress-test with scenarios.** Invent concrete edge-case scenarios that force precision about the boundaries between concepts before any code is designed around them.
5. **Only then propose 2–3 approaches** with trade-offs and a recommendation. Decisions that are hard to reverse and came from a real trade-off get their rationale recorded against their D-number in `design.html` — a future reader should never wonder "why did they do it this way?".

Do not present a final design until the interview has reached shared understanding — concretely, until every exit criterion holds:

- **Every acceptance point names its verification mechanism** — a check the developer can run (Step 4 turns these into the Verification Plan), a Figma nodeId, a measurable threshold, or an explicit "QA verifies" flag carried into the Shortcut decision summary. A criterion nothing can verify is renegotiated now, not discovered at Step 7a.
- **Behaviors carry concrete input → expected-output examples** (edge and error cases included) from the scenario stress-tests, ready to become Verification Plan criteria and the examples QA reads in the Shortcut decision summary — the human's examples are the independent source of truth every verification asserts against.
- **A non-goals / out-of-scope list is recorded in `design.html`** — prohibitions are executable; "keep it minimal" is not.
- **Known goal conflicts have a pre-ranked priority** ("when X conflicts with Y, prefer X"), captured with the relevant D-decision.
- **Qualitative goals are converted** to a measurable threshold or explicitly marked "no threshold — human judges in QA"; never left as adjectives.
- **Everything decided in the interview is written into `design.html`.** A decision that lives only in the conversation was not decided — compaction erases it, and Invariant 5 re-anchors on the artifact, not the chat.

Explore the codebase, discuss trade-offs, and write up the design. Before it goes to the human it must survive the **Design Challenge** below; only the folded-in version is presented. The gate then closes on two things, not one: the human confirms the design **and** the human explicitly declares they have no more implementation questions (**Implementation Q&A** below).

**Grounding rule — verify, don't remember.** Every type, file, field, wire format, or claim about existing behavior that appears in the design MUST be verified against the actual codebase before it is presented — open the code and cite the source path in `design.html`. This is the `figma-driven-implementation` discipline applied to code facts: query, don't remember. A design that names a type or API you haven't opened this session is not ready to present. When replacing an existing subsystem, the source implementation is the spec — read and enumerate its guards/branches/semantics before designing the replacement (see `writing-plans` "Migration discipline").

**Assumptions & Risks — mandatory section in `design.html`.** Any load-bearing claim you could NOT verify from code (third-party SDK behavior, backend contract, upgrade-state semantics) goes here: the assumption + how it will be verified (read SDK source / ask human / spike task). Do not pad the list — if everything was verified, write "None". An unverified assumption the design depends on is a risk, not a footnote.

Save output to `docs/plans/<feature-name>/design.html`.

**Code graph (draw it when the shape carries information).** Produce a code-graph architecture page whenever the feature's shape says something prose can't show at a glance: several interacting types across layers; a chain across components (entry → processing → local store → network → read-back); a fan-in / fan-out or source-of-truth merge; a local↔network (or sync↔async) boundary crossing; or guarded call paths where a condition changes the route (the page's key-flows section carries these). **Skip** it for a single function or component, a static data model or contract (use a table), pure-UI layout (Figma owns that), and trivial linear two-step calls (a sentence suffices). Author it **at design time — before the build**: draw the module as the design will shape it, existing types plus the new/changed ones, so it is the architecture spine the execution plan slices into tasks; after build, update the page and its commit note rather than leaving a stale drawing.

Follow the conventions in `references/code-graph.md` (in this skill's directory): a self-contained HTML page under `docs/plans/<feature-name>/architecture/`, linked from `design.html` — a layer-grouped inline-SVG relationship diagram (boxes coloured by architectural role only, the three-line-style arrow convention, legend), per-type purpose + public-interface tables, and a key-flows section. The SVG is hand-authored to that file's rules (full type names on one line, no overlapping text, nothing overflowing); no external diagram skill, no Mermaid/CDN. Types the design adds or changes carry a plain-text `new` / `changed` tag in their box name — text only, colour stays role and dashed border stays outside-module — and the page notes the baseline commit the design was drawn against.

### Design Challenge (devil's advocate) — mandatory

**Type: Rigid gate. Do not skip, do not rationalize past it.** Once `design.html` is written and **before it reaches the human**, the design MUST survive an adversarial review — by Codex when available, by a fresh-context subagent otherwise. The human gate reviews the *challenged* design, never the first draft.

1. Open a Codex session with the Codex CLI: `codex exec --sandbox read-only -C <project root> -o <reply file> "<brief>"`, continuing the same thread with `codex exec resume --last`. The challenger needs the real code to check the design against, and reads `design.html` by path.
2. Send `design.html` (plus the story-analysis output, or the raw request when there are no stories) and frame Codex as a **picky devil's advocate** whose only job is to attack the design. Ask it to find:
   - **Ungrounded claims** — every type, file, field, wire format, or existing-behavior claim in the design, checked against the actual code. This is the highest-value find: the author cannot reliably self-check what they believe they verified.
   - **Wrong decisions** — a rejected alternative that is actually better, a D-decision whose stated trade-off doesn't exist, a rationale that is post-hoc justification
   - **Missing Assumptions & Risks** — load-bearing claims presented as fact that were never verified against anything
   - **Unverifiable acceptance** — acceptance points naming no verification mechanism, qualitative goals left as adjectives, absent non-goals, goal conflicts with no pre-ranked priority
   - **Uncovered paths** — error/failure handling, concurrency, migration/rollback, and regressions of existing behavior the design never mentions
3. Evaluate each challenge with project context — accept real findings, push back (with reasoning) on speculative ones. **Push back hardest on scope creep**: an adversary told to attack a design reflexively proposes more architecture. The bar is "this design is *wrong or incomplete*", never "this design could be grander" — simplicity survives unless the challenger names a concrete failure it causes. Iterate in the same thread (`codex exec resume --last`).
4. Fold the agreed findings into `design.html`. Only then present the design to the human, and say what the challenge changed.

**Codex CLI unavailable → internal fallback, not a stop**: hand the same brief to a **fresh-context subagent** (`general-purpose`, read-only) that took no part in producing the design. Evaluate and fold its findings exactly as in steps 3–4, and note in the summary that the challenge ran on the internal fallback. The gate is on the adversarial pass, not on Codex specifically — it is never skipped.

| Thought | Reality |
|---|---|
| "The grilling interview already stress-tested this" | The interview tested the human's intent. The challenge tests the *design against the code* — a different axis. |
| "I verified every claim as I wrote it" | Self-verification is exactly what this catches; you cannot audit your own recall. |
| "The design is small / obvious" | Then the challenge costs one round and returns nothing. Run it. |
| "Auto mode means I can skip this" | Auto mode does not disable a Rigid gate. |

### Implementation Q&A — the gate closes only after this

**Type: Rigid gate.** After presenting the *challenged* design and what the challenge changed, ask the human — explicitly, in the same turn — whether they have questions about how anything gets implemented: "how does X actually work", "where does Y live", "what happens when Z". Never run this on an unchallenged draft.

- **Answer under the Grounding rule.** Open the code and cite `file:line`; an answer naming a type, file, or API you have not opened this session is not an answer. Fan out read-only agents for the legwork — the answer itself is given inline (Invariant: gates are never delegated).
- **Fold back before continuing.** Any answer that exposes a gap, changes a decision, or surfaces an unverified assumption goes into `design.html` (new or changed D-number, or Assumptions & Risks) *before* the round can end. Re-present what changed, then ask again. Nothing else is written down — the Q&A itself produces no artifact.
- **Loop until the human explicitly says they have no more questions.** Then share the decisions with QA (below) and proceed to Step 4.

| Thought | Reality |
|---|---|
| "They approved the design, so the round is closed" | Approval and "no more questions" are two separate confirmations. Both are required. |
| "They went quiet / said 'looks good'" | Silence and vague praise are not the declaration. Ask again. |
| "This question is too deep for design — it's an implementation detail" | If it can't be answered from the code now, it's an assumption or a risk. Record it as one. |

### Share decisions with QA (Shortcut)

QA owns the test cases and works with their own method, usually after dev has started — so dev and QA can read the same story two ways. Closing the gate therefore includes posting the agreed decisions onto the Shortcut stories, so QA writes their cases from the same understanding dev is building from. Skip this when the feature has no Shortcut stories (Step 2 was skipped).

- **One comment per story**, covering what applies to that story, in English. Content is *decisions, not thinking*: the behavior-affecting D-decisions in plain product language (no class or file names); out of scope / non-goals; the concrete input → expected-output examples, edge and error cases included; the edge cases and risk areas dev considered; and open questions still waiting on PM. Leave out the exploration, the alternatives weighed, and implementation detail.
- **The human approves the draft before anything is posted** — show it in the same turn as the closing gate; a story comment is visible to the whole team. Then post with `mcp__shortcut__stories-create-comment`. Shortcut MCP unavailable → hand the approved text to the human to paste.
- **Divergence goes to PM, not to a guess.** If QA (or a story comment) reads the story differently from a posted decision, neither side's reading wins — ask the human to take it to PM, and record the answer as a changed D-decision.

---

## Step 4 — Test Plan [human gate]

**Skill**: `fatsecret-workflow:write-test-plan`
**Condition**: Always (Full Track).

Uses the confirmed design and the story-analysis output (if available) as input. Produces two artifacts with two owners:

- **Draft QA test plan** — manual cases written from the confirmed design, saved as `docs/plans/<feature-name>/test-plan.html` and, after approval, uploaded to TestSecret as **Draft**. A head start for QA, not the final word: from upload on QA owns the cases and reviews, edits and promotes them.
- **Verification Plan** — the developer's own test considerations, appended to `design.html`: one row per deliverable — criteria (the observable result that proves it) → method (script+fixture in `verification/<feature>/` / review-only / simulator check / unit test for wrong-answer-is-a-bug logic / integration test for persistence contracts). **The human confirms it as one table in one batch** — the criteria and method per deliverable are the human's call, corrected per row in their reply, never asked one row at a time.

Before either is finalized, `write-test-plan` runs a **mandatory devil's-advocate coverage challenge** over both — Codex when available, a fresh-context subagent otherwise; the gate itself is not skippable. One human gate covers both; the upload follows approval.

Wait for human confirmation before proceeding.

---

## Step 5 — Execution Plan [human gate]

**Skill**: `fatsecret-workflow:writing-plans`
**Condition**: Always (Full Track).

Uses all prior outputs (story-analysis, design including its Verification Plan) as input.

**Default: an Execution Checklist, not a separate document.** For most features, `writing-plans` produces a lightweight checklist appended to `design.html` — one entry per task: title, exact Files (+ Figma nodeId for UI files), the Verification rows it owns (from the Verification Plan), its acceptance point, and the standard Implement → Verify → Review → Commit steps. **No code blocks** — implementers read the real files at task time.

**No separate plan document.** A task that will be dispatched to a subagent or performs a destructive migration carries its extra detail inside its own checklist entry — see `writing-plans` "One Artifact".

**Folder placement for new files** is confirmed with the human ONCE, as a single batched `AskUserQuestion` (each new file: proposed folder + one-line rationale) — not per file.

**Figma coverage:** every UI task must carry its Figma nodeId in its checklist entry. If a UI element has no design node, ask the user for the design — or record an explicit waiver — before the checklist is finalized. (Implementation-time enforcement is Invariant 3, so this holds even when this step's documents are skipped.)

Wait for human confirmation before proceeding.

**Commit the plan artifacts before Step 6.** Once the checklist is approved, ask for the OK (Invariant 1) and commit the feature's plan folder as its own docs commit — otherwise Step 6's clean-tree checks can never pass. Plan-doc edits made later (design re-entry, 7a deferrals, the code-graph update) are committed the same way.

---

## Step 6 — Build

**Condition**: Always.

### Pre-task check

Before starting each new task, run `git status` to confirm the prior task committed cleanly. If uncommitted changes remain, investigate the root cause before starting the new task.

**Task brief — announce before touching anything.** Open every task by presenting a short, visually prominent brief (bold header + bullets) restating its checklist entry — never invented, never expanded:

> **▶ Task N — \<title\>**
> - **Goal**: the observable outcome this task delivers — its acceptance point
> - **Files**: the declared Files list (+ Figma nodeId for UI files)
> - **Verification**: the entry's Verification rows (criteria + method), or "simulator check + review — pure UI"; plus its Re-run list

This is the checklist entry re-read aloud, not a re-plan: it re-anchors on the artifact at every task boundary (the same discipline Invariant 5 applies after compaction) and gives the human a glanceable checkpoint — a surprising brief gets caught before code is written, not at review. If what you were about to do doesn't match the brief, stop and re-read the checklist instead of improvising. Delegated tasks carry the same brief at the top of the dispatch prompt.

### Execution

Execute each task exactly as written in the execution checklist. **The checklist is the single source of truth for the steps inside a task.**

Invariants Step 6 enforces on top of the checklist:

- `review-task` performs ONE review pass per invocation and returns its report. **The caller (you) owns the review-fix loop**: fix the reported issues (fixes only — no commit), re-invoke `review-task`, at most 3 rounds; after 3 rounds without an APPROVE recommendation, treat the task as `ESCALATED`.
- `review-task`'s `APPROVED` verdict unlocks the task's commit + `TaskUpdate(completed)`. On `ESCALATED`, halt the task and surface the issues to the human.
- After `APPROVED` and before the commit, run the **Knowledge Delta Check** (defined in `writing-plans`): only if the review surfaced a human correction about how this codebase works, or a conflict with `docs/plans/CODEBASE-KNOWLEDGE.md`, propose the addition/fix to the human (write only on an explicit yes); otherwise skip silently. Never blocks the commit.
- **Commit reliability:** stage exactly the task's declared Files (`git add <paths>` — never `git add -A` / `git add .`); before committing, verify staged paths ⊆ the task's Files list and question anything extra; after committing, run `git status` and confirm the tree is clean before declaring the task complete.

### Design change mid-build (re-entry procedure)

When the design changes after tasks have been built (a new meeting decision, a revised backend contract):

1. Update `design.html` first — new/changed D-numbers — and diff the old vs new decisions.
2. List every already-completed task the delta affects.
3. Confirm the list with the human, then fix the affected completed tasks FIRST (each fix passes the normal review → commit gate) before building new tasks.
4. If a changed decision was in the Shortcut decision summary, post an update comment on the affected stories saying what changed (same approve-before-post rule) — QA may already be writing cases from the old one.

---

## Step 7 — Verify & Ship

Two sub-steps to close out the feature. **Evidence before claims** applies to both: any completion claim ("verified", "build succeeds", "bug fixed") is made only after its verification command has run *in this session* with output confirming it — no "should pass", no extrapolating from an earlier run, no trusting a subagent's success report without reading the diff.

### 7a. Deliverable verification [blocking]

**Skill**: `fatsecret-workflow:validate-test-plan`
**Condition**: Always (Full Track).

7a does not re-run what Step 6 already proved. It validates what no task verified: acceptance points in `design.html` that no Verification Plan row covers, rows left UNVERIFIED or punted at a task's review gate, and the feature's cases in TestSecret (the uploaded draft, as QA has since edited it). Rows a task's review already verified are cited from that review's evidence, not run again; when nothing is left to validate, say so in one line and move on. The validation is refute-first, typed verdicts per sub-clause, runtime/external clauses punted explicitly rather than graded from static reading. **Blocking:** unresolved acceptance gaps stop the workflow here — do not proceed to 7b unless the user explicitly defers a gap (record the deferral and reason in `design.html`); each punt-list item needs the human's do-or-waive call. For features whose per-task review gates were structurally weak — no Figma AND no unit-testable logic (e.g. backend-integration work) — 7a must include verification against the real backend/integration (the skill's runtime pass), or an explicit user waiver.

When the TestSecret cases were validated, after the human resolves the report the skill records the outcome to TestSecret (Phase 5): a run whose per-case results carry the verdicts (`Dev-Verified` / `AI-Tested` / `Failed` / `Retest` / `Skipped`), with punted cases left `Untested` — that Untested set is QA's manual queue.

### 7b. Finish branch

**Condition**: Always.

With 7a resolved, determine the base branch (the branch this work forked from — typically `main`; confirm with the human if in doubt), then present exactly these options and execute the choice (merge/push still follow Invariant 1 — the human's same-turn choice is the OK):

> Implementation complete. What would you like to do?
> 1. Merge back to `<base-branch>` locally
> 2. Push and create a Pull Request
> 3. Keep the branch as-is (I'll handle it later)
> 4. Discard this work

- **1 — Merge:** checkout base → `git pull --ff-only` (a pull that would create a merge commit stops here — surface it and ask) → merge → re-run the feature's declared verification on the merged result (its test classes scoped + its `verification/<feature>/` scripts) → only then clean up: worktree first (rules below), then `git branch -d`.
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
