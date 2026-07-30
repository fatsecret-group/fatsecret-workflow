---
name: edge-case-hunt
description: Use after story analysis and before design, when a feature's stories and designs are understood but the gaps in them are not — runs fourteen systematic probes against the codebase to find the edge cases, contradictions and phantom identifiers that specs leave out, then classifies each finding and drives the product ones into a decision log that outranks the story text
---

# Edge-Case Hunt

## When to Use

- **Step 2.5 of `feature-workflow`** — after `story-analysis` produces Items, before `write-test-plan` and before design.
- Standalone, whenever you are handed a feature spec and asked "what's wrong with this?"
- Before quoting an estimate on work whose spec you did not write.

**Do not use it** as a code review (`review-task`), as a bug hunt in shipped code, or as a
substitute for reading the stories.

## Why this exists

A story tells you what to build when everything goes right. Roughly half the implementation
decisions are about everything else, and almost none of them are in the story. Left undiscovered
they surface as: a field name that does not decode, a sheet that never shows again, an experiment
that never activates, a loading state that renders as "locked".

`story-analysis` already asks for a picky mindset and an "Ambiguities & Open Questions" section,
but gives no method — so what gets found depends entirely on who is running it. The probes below
are that method. Each is a question that has repeatedly returned a real finding.

**The output is not a list of worries. It is a decision log that outranks the story text.**

> The examples throughout are real, from one iteration (Smart Food Scan free-trial experiment)
> where these probes produced roughly forty product decisions and four latent defects before a
> line was written. Where an example was found on the Android side of a shared feature, it says
> so — those are the ones worth checking hardest here, since the same spec produces both apps.

---

## The core move: three-way cross-check

Never compare two things. Compare three:

```
        what the STORY says
                 ↕
      what the DESIGN shows   ←→   what the CODE already does
```

Two-way comparisons confirm; three-way comparisons contradict. Almost every finding below came
from a place where two sources agreed and the third did not — and the two that agreed were
usually the two nobody had checked against reality.

**Rule: a claim is not established until you have seen it in the third place.** The story names
an event → grep for it. The design shows a grey block → find the token. The code has a premium
guard → check the story wants one.

---

## First: does the other platform already have a decision log?

**Check before you probe anything.** Most stories are written once and cloned to both platforms,
so the same ambiguities land on both desks. If the other platform ran this hunt first, they have
already taken those questions to the PM and written the answers down.

```bash
# the other platform's repo, feature folder
ls  <other-platform-repo>/docs/plans/<feature>/decisions.md
```

### If it exists, it is an input — and it is authoritative

**Do not re-derive it, and do not re-ask the PM questions it already answers.** A PM asked the
same question twice gives two answers, and now the platforms are divergent on a point that was
already settled.

Treat it as a spec that outranks the stories, exactly as its own header says. Your job is then
three things, in order:

1. **Verify each decision is implementable here.** The decisions are product-level and
   platform-neutral by construction, but their *mechanics* are not — P3 (unknown state), P9
   (evaluation time), P13 (appearance multiplicity) and P14 (device-variant split) all behave
   differently here. A decision like "show a skeleton while the status is unknown" assumes the
   screen *has* an unknown state to render; confirm it does.
2. **Run the probes anyway, scoped to what is platform-specific.** The shared product questions
   are answered; the platform-specific findings are not, and nobody else will find them. Anything
   you find that changes a shared decision goes back to the PM as an amendment, not a new log.
3. **Add your findings to §10 of the same document**, the "platform-specific implementation
   notes — do not copy" section. Never start a second document.

### If it does not exist, you are producing it

Write it for both platforms from the start — that is what §10's quarantine is for. When you hand
it over, say explicitly that it outranks the story text, and which story sentences it supersedes.

### The failure mode this prevents

**Two decision logs for one feature.** They start identical, diverge within days, and the person
who notices is a tester filing a bug that is correct on one platform and wrong on the other. One
document, one location, both platforms reading it.

Concretely, from the iteration these examples come from: the log forked into two copies when one
was moved to a desktop, and the copy that was about to be handed to the other platform sat five
decisions stale — including the API field name (P1) and the remote-config key. Keep it in the
repo, one copy, and re-verify before handing it over.

### Sync protocol

| Section | Rule |
|---|---|
| §1–§9 product decisions | **Identical on both platforms.** A change on one side is a change on both — say so in the same message |
| §10 platform notes | Diverges freely; marked "do not copy" so the document stays handable |
| §11 superseded story text | Shared — the stories are shared |
| §12 open items | Shared, grouped by owner; add a row for anything needing agreement from *both* platforms rather than from a person |

---

## The probes

Run all of them. They are ordered so earlier ones feed later ones. Each has a **question**, the
**mechanics** for answering it here, and a **real finding** it produced.

`P1`–`P12` are shared with the Android workflow's counterpart skill and are deliberately numbered
the same, so a finding can be cited across a shared decision log — "P7 on both platforms" has to
mean the same thing. `P13`–`P14` are iOS-only.

### Group A — Reality checks: does what the spec names actually exist?

#### P1 · Identifier reality check

> **Every name the spec utters must be grepped before it is believed.** Event names, API field
> names, remote-config keys, `FSString` keys, endpoint paths, screen names.

Specs are written in prose by people who are not looking at the code. Names drift, get renamed,
or were never real.

```bash
grep -rn "<identifier>" --include='*.swift' --include='*.m' --include='*.h' .
# localizable keys live in Localizable.strings, referenced as
#   FSString("key")     (Swift)
#   FSString(@"key")    (ObjC)
```

Three distinct failure modes, all common:

| Mode | What you find | What it means |
|---|---|---|
| **Phantom** | zero hits anywhere | the name does not exist — the spec invented it |
| **Renamed** | zero hits, but a near-miss exists | the spec is out of date; find the real name |
| **Right name, wrong plumbing** | hits, but wired elsewhere | see P1b |

> **Real finding.** The Epic and two stories named `page_view_food_launcher` as the A/B
> activation event. **It does not exist.** The real event is `page_view_food_search_launcher`
> (`FoodLog/FoodLogLauncher.swift:24`). The Firebase experiment would have been configured
> against a name that never fires — no enrolment, silently, presenting as "nobody is qualifying".

> **Real finding.** Every story said the remaining-count field was `scans_remaining`. The API
> returns `remainingUses`. The `Codable` model would have failed to decode the one field the
> entire feature is built on.

#### P1b · Destination / wiring check

> **An identifier existing is not the same as it being usable for the purpose the spec assumes.**

Analytics is where this bites. There are two independent destinations, and an event on one is
invisible to the other:

| Path | Call | Ends up in |
|---|---|---|
| Avo | `avo?.someEvent(...)` | RudderStack |
| Firebase | `FSAnalytics.logFirebaseEvent(_:params:)` | Firebase Analytics |

Firebase A/B activation events **must** be on the Firebase path.

> **Real finding.** `pageViewFoodSearchLauncher` exists and fires — but only via Avo →
> RudderStack. It is absent from Firebase Analytics, so it cannot be selected when configuring
> the experiment. The resolution came from P10.

The same question applies to: remote-config keys (is there a registered default in `RCValues`?),
strings (is the key in the shipped `Localizable.strings`, or only on a translation branch?), and
endpoints (does the networking layer already attach the auth headers this one needs?).

#### P11 · Term collision

> **Grep the domain term and read *all* the definitions, not the first one.**

Features accumulate. The same word often already means something else in the same area of code.

> **Real finding.** The new feature has variants "A" and "B". A *previous* experiment on the same
> screen also names its arms "Variant A"/"Variant B" — **and they are inverted** relative to the
> new ones. Every sentence containing "variant B" in that file is ambiguous.

This codebase has an extra source of collision: **class names do not track the product name, and
some carry historical typos** (`AIAssitant*`). Grep the product noun, not the class you expect to
find.

Also run it on the feature's own nouns ("scan", "trial", "count") and on any verb the story uses
as though it were precise ("dismiss", "reset", "expire").

#### P12 · Design-file forensics

Design files mislead in specific, repeatable ways. Check all four:

| Trap | How to catch it |
|---|---|
| **Layer name ≠ rendered text** | Never read copy from the layers panel. `get_screenshot` every frame you quote |
| **Half-updated revisions** | Find sibling frames that should differ by exactly one variable. More than one ⇒ one was not updated |
| **Stray annotations** | Notes not bound to any node. Verify against the node's actual fills before implementing |
| **Two files, two truths** | A FigJam board for flows and a design file for visuals will disagree. Write down which wins for which question |

> **Real finding.** A FigJam layer named "Take Photo" rendered as "Smart Food Scan". Reading the
> layers panel would have shipped the wrong label.

> **Real finding.** A copy revision was applied only to the "5 left" frames; the matching
> "1 left" frames kept the old strings. The pair differs only by a number, so the copy has no
> reason to differ — a design oversight, caught by comparing siblings. Without it the app would
> have shown different copy at 5 remaining and at 1 remaining.

> **Real finding.** A handover annotation read `background gradient: amber.0 -> amber.10` while
> the node's actual fill was a flat token. Implementing the annotation would have produced
> something the design never showed.

---

### Group B — State-machine probes

#### P3 · The unknown state

> **What does this screen render between "appeared" and "data arrived"? Does that state exist in
> the code at all?**

The usual answer: it renders the *false* branch of a synchronous read of asynchronous truth.
Users see a confident wrong answer.

Look for:
- Synchronous statics read in `viewWillAppear` / `viewDidLoad` / a SwiftUI `body`: `IAPHelper.isPremium`, `RCValues.shared.*`
- A companion "loaded" flag that exists but is never consulted
- `@Published` / `@State` initial values indistinguishable from real ones (`false`, `0`, `[]`) rather than `nil` or a `Loading` enum case

```bash
grep -rn 'IAPHelper.isPremium\|RCValues.shared' --include='*.swift' --include='*.m' <feature dirs>
```

> **Real finding (Android side of a shared feature — check the equivalent here).** The food
> launcher read premium status synchronously from a singleton and **never consulted its
> "status loaded" flag**, so "we don't know yet" rendered identically to "not premium". With a
> paid feature about to be gated on that boolean, the unknown state needed a real representation
> — which became a **product** decision (show a skeleton), not an implementation detail.
> `IAPHelper.isPremium` is the same shape here: a synchronous static with no loading state.

Ask it for each of: **premium status, server-side quota, remote-config assignment, network
result.** Each has its own unknown window, and they close at different times.

#### P4 · State versus transition

> **Is this trigger an edge or a level?**

A condition like `remainingUses == 0` is a **state**, and states are permanent — anything driven
off it fires on every appearance, forever. "The moment it became 0" is a **transition**, and
needs a one-shot.

Getting this wrong produces either a sheet that appears once and never again when it should
recur, or one that appears forever when it should appear once.

> **Real finding.** The trial-expired page is specified as appearing when the last scan is used.
> Driven off `remainingUses == 0` it would have re-appeared on every subsequent Diary visit for
> the rest of the user's life. Driven off the 1→0 transition it needs no persistent "already
> shown" flag at all — the transition happens once by construction.

Here this compounds with **P13**: a level-triggered presentation in `viewWillAppear` re-fires
every time a pushed or presented controller is dismissed, not merely on each navigation in.

The follow-up is the valuable part: **if it is a transition, what happens to a user who misses
it?** (Backgrounded, killed, hit it on another device.) Separate product decision; the story will
not have it.

#### P8 · Upgrade & first-encounter path

> **For every new persisted value: what is it for a user who updates the app mid-flight?**

New `UserDefaults` keys read as `nil` / the supplied default for every existing user. Decide
explicitly what that means — it is almost always a product question disguised as a nil check.

```swift
UserDefaultsManager.getIntSetting(key, defaultValue: 0)   // is 0 "none left" or "not fetched yet"?
```

| New value | The question |
|---|---|
| A counter | does absent mean "unlimited", "zero", or "unknown"? |
| A "has seen X" flag | are existing users treated as having seen it, or not? |
| A cached server value | does the feature lock or unlock while it is unknown? |

> **Real finding.** The local quota cache is optional. Absent could reasonably mean "not fetched
> yet, allow" or "no quota, lock". The decision (**lock**, plus a skeleton while unknown) changed
> the design of two launcher entries.

Also covers users who already performed the gated action before the feature existed, and users
mid-flow when the app updates underneath them.

#### P9 · Evaluation-time audit

> **When is this value read — once at init, lazily on each access, or observed?**

A correct value read at the wrong time is a bug that presents as a data problem.

```swift
let flagX = RCValues.shared.bool(forKey: "x")          // snapshot at init, never updates
var flagY: Bool { RCValues.shared.bool(forKey: "y") }  // re-read on every access
```

Check `RCValues.swift` for which form the flags in your area use, and check view models for
premium/account state captured at init.

> **Real finding.** A new experiment variant key evaluated at init is read before the remote
> fetch completes and never re-read, pinning every user to control. It must be computed.

> **Real finding.** Premium state in the affected screens is a one-shot snapshot. After an
> in-flow purchase the underlying screens keep stale state — a pre-existing refresh bug the
> feature would have inherited and been blamed for.

---

### Group C — Collision probes

#### P2 · State ownership collision

> **Before introducing any new counter, flag or cache: who already owns something that means
> almost this?**

Two pieces of state with overlapping meaning drift apart. The bug appears months later and is
nearly impossible to attribute.

```bash
grep -rn 'UserDefaultsManager\.\(get\|set\)' --include='*.swift' --include='*.m' . | grep -i <feature>
```

> **Real finding.** A local counter `keyNumberOfSurveysShown-SmartFood`
> (`SmartFoodSurvey.swift:83`) already existed, serving only the feedback-survey cadence. The new
> feature needed "how many scans remain", from the server. The names are near-synonyms and the
> story explicitly warned about "count drift". Decision: they never touch each other.

**Also ask the reverse: does the new feature change the meaning of the existing state?** Here it
did — trial scans would have polluted the survey cadence. This codebase happened to be safe,
because the premium guard at `SmartFoodSurvey.swift:87` sits *before* the increment at `:91`.
That is worth knowing deliberately rather than by luck: the Android equivalent had no guard at
all and would have shown the survey to trial users on their first scan.

#### P5 · Moment contention

> **Enumerate everything that already fires at the moment your new thing fires.**

Common moments: returning to the Diary after a save, `applicationWillEnterForeground`,
`viewWillAppear`, post-purchase.

For each competitor, decide the outcome — and note that "don't show it" has three very different
implementations:

| Outcome | Meaning |
|---|---|
| **Dropped** | skipped this occasion, shown at the next natural opportunity |
| **Deferred** | queued, shown immediately after |
| **Lost** | never shown again — usually unintended |

> **Real finding.** Four prompts compete for "returned to the Diary after logging food":
> reminders opt-in, privacy settings, the feedback survey, and the new upsell panels. Deciding
> priority was easy. The trap was underneath: one prompt's condition is an **exact equality** on
> a counter that increments on every save (`count == 2`). "Drop it this time" therefore means
> **lost forever**, not deferred — the exact opposite of the intent. The fix is relaxing to `>=`
> with the existing already-shown flag preventing repeats.

**Generalise it: whenever a decision is "skip it this time", go and read the skipped thing's
trigger condition.** Exact equality against a monotonic counter is the tell.

#### P7 · Gate axis audit

> **Read what an existing gate actually keys off — not what it is named, nor what everyone says
> it does.**

New screens frequently land inside or outside an old gate by accident, because the gate keys off
something structural (a mode, a controller, a tab) rather than semantic.

> **Real finding.** An existing auto-upsell is gated on being in the camera/photo mode. The new
> trial camera **is that same mode**, so the new screen would have inherited the old upsell — two
> sheets stacking. The natural-sounding fix ("only in variant A") is wrong: it reads the variant
> to answer a question that is not about the variant. The correct condition is **"not in trial
> mode"**, which satisfies every case without reading the variant at all.

The pattern: when a gate condition mentions your experiment's variant, suspect it. Variants are
usually a proxy for a real condition; name the real one.

---

### Group D — Failure-path probes

#### P6 · Reachability of the new failure

> **Trace your new failure condition through the *existing* error handler. Where does it land
> today?**

New features add new ways to fail. Those failures flow into error handling written before the
feature existed, and usually land in a generic bucket with the wrong message.

Find the request's error path and enumerate what is handled explicitly versus what falls through
to the default.

> **Real finding.** Quota exhaustion surfaces as **HTTP 403**. The 403 plumbing already exists,
> but on the scan path it falls through to the generic "AI service unavailable" state. Users who
> ran out of free scans would have been told the AI was broken.

Also probe the **success path's missing branches**:

> **Real finding.** A success handler shaped `if items?.isEmpty == false { … }` with no `else`.
> A successful response containing zero items leaves the screen loading forever. Pre-existing,
> found by reading the success handler rather than the spec.

**Checklist per new network call:** empty-but-successful response · each documented error code ·
undocumented codes · timeout · offline · a response arriving after the controller is gone.

---

### Group E — Precedent probes

#### P10 · Precedent forensics

> **When the spec (or you) says "do it like X did", open X's actual code. On both platforms.**

Precedents are cited from memory, and memory is wrong. Worse, the precedent itself may be
divergent — in which case copying it propagates the divergence.

> **Real finding.** The activation-event deadlock was solved by a shipped experiment (sc-61235)
> which created a **dedicated Firebase event** precisely because the Avo event was not in
> Firebase (`SmartFoodScan.swift:750`). Reading that story gave a working pattern and its
> rationale, turning an apparent deadlock into a one-line decision.

> **Real finding, and the reason this probe says "both platforms".** That same precedent event is
> implemented **differently on each side**: here it passes an `origin` parameter and guards with
> `hasFiredTakePhotoActivation` so it fires once per controller lifetime; Android passes no
> parameters and has no guard, firing on every entry to the photo tab. Android's volume is
> therefore systematically higher, so any metric using the event as a denominator is not
> comparable across platforms. Cloning the pattern without reading both sides would have
> reproduced the split in the new event.

Where precedents live: shipped Shortcut stories (`stories-search` with `isDone: true`), git
history for the file, `docs/plans/CODEBASE-KNOWLEDGE.md`, `docs/plans/<feature>/`,
`docs/maestro/<feature>/`, and the other platform's source.

> **On reading the other platform.** Most stories are written once and cloned to both platforms,
> so the same ambiguity lands on both desks. Any divergence found here is worth one message
> rather than two independent guesses — and if it is found *after* both sides have built, it is
> no longer a question but a defect. This is the cheapest moment.

---

### Group F — iOS-specific probes

These have no Android counterpart: the equivalent failure either cannot happen there, or fails in
a way the shared probes already catch.

#### P13 · Appearance multiplicity

> **`viewWillAppear` is not "the user navigated here". It fires on every appearance** — including
> returning from a pushed controller, dismissing a modal, and returning from the app switcher.

Anything one-shot placed there fires many times. Anything counted there over-counts.

> **Real finding.** The precedent activation event is fired from *two* call sites
> (`viewWillAppear` and the barcode→scan toggle) and is only correct because an explicit
> `hasFiredTakePhotoActivation` instance flag collapses them. Remove the flag and the event
> multiplies. When cloning that pattern, **the flag is the load-bearing part, not the call.**

Ask, for every presentation, event and fetch you add: **is once-per-appearance right, or
once-per-navigation, once-per-session, or once ever?** Those are four different implementations
and the story will specify none of them.

Related: `applicationWillEnterForeground` fires after permission dialogs and share sheets, not
only after real backgrounding — so "on foreground" work runs more often than the story imagines.

#### P14 · Device-variant file split

> **A screen is often three files plus two `.xib`s. A change made in one is silently absent from
> the others.**

```
Foo.m            shared logic
Foo-iPhone.m     phone layout / behavior
Foo-iPad.m       tablet layout / behavior
Foo-iPhone.xib   Foo-iPad.xib
```

This fails *silently* — the iPad path simply keeps the old behavior, and nobody looks until a
tester with an iPad does. (Android's resource-qualifier equivalent fails loudly instead.)

**For every change to a UIKit/ObjC screen: list the directory for siblings, and check the `.xib`s
for outlets the new code expects.** Also establish whether the feature is UIKit or SwiftUI before
assuming where the logic lives — older features are `.m` files with `.xib`s, and reading only the
SwiftUI file produces a confident, wrong conclusion.

---

## Classifying what you find

A finding is worthless until it is routed. Every one gets exactly one tag:

| Tag | Meaning | Goes to |
|---|---|---|
| **`[DECISION]`** | A product question the spec does not answer. Not resolvable by reading code | PM — the decision log |
| **`[ALIGN]`** | The two platforms would diverge without agreement | Both platforms — usually one message |
| **`[BUG-EXISTING]`** | Broken today, independent of this feature, but sitting in the path | Flag it; decide in/out of scope explicitly |
| **`[IMPL]`** | Real, but you can just decide it. Record the choice and move on | The design doc |
| **`[NOTE]`** | Historical divergence nobody is going to fix this iteration | Write it down so it is not re-reported |

**The most common failure is over-tagging `[DECISION]`.** If you can answer it by reading the code
or by picking the obviously-right option, it is `[IMPL]`. A PM handed thirty questions answers
none of them well; handed eight, they answer all eight.

**The second most common failure is under-tagging `[BUG-EXISTING]`.** Pre-existing bugs in the
path you are about to touch will be attributed to your change. Surface them, then decide in-scope
or out-of-scope *out loud*.

---

## Driving decisions out of the PM

The findings are only half the work; unresolved ambiguity is worth almost nothing.

1. **One question at a time.** A bulk list gets skimmed and half-answered. Ask, wait, then ask the next.
2. **Every question carries options and a recommendation.** "What should happen when X?" gets a shrug. "When X we can do A (consequence) or B (consequence) — I'd suggest A because Y" gets a decision in one exchange.
3. **State the cost of each option in user-visible terms**, not implementation terms.
4. **Record the answer as authoritative immediately**, in the decision log, with a number.
5. **When you discover a question rested on a wrong premise, say so and re-ask.** An answer to a mis-framed question is worse than no answer, because everyone will believe it.
6. **Batch the cross-platform items into one message**, not one per finding.

---

## Output

Two artifacts, both in the feature folder. The decision log is the one that matters.

### `edge-cases.md` — the hunt's raw output

One row per finding: the probe that found it · the finding · evidence (`file:line`, Figma nodeId,
story sentence) · tag · status.

Keep it. When someone asks "did we think about X?" six weeks later, this answers it in seconds.

### `decisions.md` — the authoritative log

**This document outranks the story text**, and says so at the top. Structure that has proven to
work:

```markdown
# <Feature> — Confirmed Product Decisions
**Epic** / **Iteration** / **Source** (who confirmed, when) / **Last updated**

## How to use this document
- Where this document and a story disagree, THIS DOCUMENT WINS. §11 lists the superseded sentences.
- §1–§9 are product decisions and apply to both platforms.
- §10 is platform-specific implementation notes — context only, do not copy.
- §11 is the superseded story text.
- §12 is what is still open, by owner.

Status markers: ✅ decided · 🔶 interim (will change) · ❓ open

## 1..9 — decisions, numbered, one table row each
## 10 — platform-specific notes (marked "do not copy")
## 11 — Story text that is now superseded    ← the section people forget, and the one QA needs
## 12 — Still open / pending, BY OWNER
```

Four properties that make it work:

- **Numbered decisions** (`1.3a`, `7bis.4`) so the design doc, the test plan and commit messages can cite rather than restate them.
- **§11 exists.** The Shortcut stories usually do not get edited. Without an explicit "these sentences are dead" list, QA verifies against superseded text and files bugs against correct behavior.
- **§12 is grouped by owner**, so a glance says who is blocking.
- **Platform-specific notes are quarantined in one section** and marked "do not copy", so the whole document can be handed to the other platform as-is.

**Keep it in the repo**, not in a chat thread or on someone's desktop. A decision log outside
version control has no history and no review, and quietly forks into two copies — ours did, and
the stale copy was the one about to be handed to the other platform.

---

## Anti-patterns

| Anti-pattern | Why it fails |
|---|---|
| Trusting an identifier because the story is confident | Stories name things that do not exist (P1) |
| Reading copy from the Figma layers panel | Layer names are not rendered text (P12) |
| "The spec doesn't mention it, so it's out of scope" | The gaps *are* the deliverable here |
| Reporting every finding as a PM question | Thirty questions get zero answers |
| Comparing story ↔ design and stopping | Two-way agreement is the commonest way to be confidently wrong |
| Copying a precedent from memory | Precedents drift, and may themselves be divergent (P10) |
| Reading only the SwiftUI file | Older features live in `.m` + `.xib`; the SwiftUI file is often a shell (P14) |
| Putting a one-shot in `viewWillAppear` | It is not one-shot (P13) |
| Quietly fixing a pre-existing bug you found | Scope creep, and it hides a real defect from whoever owns it |
| Leaving an ambiguity as "TBD" in the design doc | It resurfaces mid-implementation at ten times the cost |
| Keeping the decision log outside the repo | It forks, and you will not notice which copy is stale |

---

## Where this sits in the workflow

```
Step 2   story-analysis      → Items, dimensions, first-pass ambiguities
Step 2.5 edge-case-hunt      → THIS SKILL: probes → edge-cases.md + decisions.md
Step 3   design              → cites decision numbers rather than restating them
Step 4   write-test-plan     → every [DECISION] becomes at least one case
```

Run it **before** the test plan: cases written against unresolved ambiguity get rewritten, and
uploading them to TestSecret makes that churn visible to QA.

Run it **before** design: a design built on an assumption that later flips is a redesign, not an
edit.
