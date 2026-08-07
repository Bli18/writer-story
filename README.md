# Writer Story (《作家物语》)

A pixel-art life simulation in which you play a novelist across fifteen years —
choosing what to write, watching it succeed or fail, and living with the career
that results. Built solo in Godot 4 / GDScript.

**Status:** playable end-to-end, in balance and polish. All 180 in-game months,
21 endings, and the full publishing economy are implemented and reachable. Steam
integration is wired; store release is pending. This repository is a write-up of
how the game was designed and built — **the source code is private** (see the
last section).

| | |
|---|---|
| **Engine** | Godot 4.6, GDScript (statically typed) |
| **Scope** | 178 scripts / ~72k lines · 45 scenes · 21 gameplay systems |
| **Content** | 60 data files · 2,231 localized strings × 3 languages (zh/en/ja) |
| **Design docs** | 20 system design documents · 10 architecture decision records |
| **History** | 1,156 commits, June–August 2026 |
| **CI** | Headless test harness on every push and PR |

---

## Screenshots

> _Placeholder — screenshots and a short gameplay GIF go in `docs/screenshots/`._

| | |
|---|---|
| _Writing screen_ | _Monthly review_ |
| _Publishing & reception_ | _Ending sequence_ |

---

## Background

This game comes from two things I have had for a long time. One is Kairosoft's
*Game Dev Story*, which I played as a kid and have never really gotten over —
the small studio, the slow accumulation, the years that only make sense looking
back. The other is writing, which I have loved for as long as I have loved
reading.

I do not write for a living, and most likely never will. That is the ordinary
case: for almost everyone who loves writing, it is the thing you do at night,
around the job that actually pays. This game is for those people. For anyone who
loves to write, cannot live on it, and still likes to imagine the version of
their life where they could — and who would rather play that life than merely
daydream it.

Which shapes the design. Most career simulations are optimization games wearing
a theme: the numbers are the content, and the flavor text is decoration. That is
the wrong instrument for this fantasy. Writing does not feel like solving for a
maximum; it feels like choosing what to be about, and then finding out what the
world does with it. So the interesting decision here is an **aesthetic** one, and
the numbers exist to give that decision consequences.

The core loop is a three-axis choice: **genre × theme × style**. Those three
axes have 532 curated compatibility rows behind them, sorted into four tiers —
_in tune_, _promising_, _bold_, _awkward_. A bold pairing is not a worse choice
than a harmonious one; it is a higher-variance one. Writing crime fiction about
rural displacement in a restrained register is a different bet from writing it
hot-blooded, and the game is built so that both bets are legible before you make
them and survivable after.

Around that sit the things that actually make a writing career: money and morale
as the only two resources, four separate reputations (readers, critics,
commercial, historical) that move independently and sometimes against each
other, a publishing form that is chosen **after** the manuscript exists rather
than before, and three long-running relationships — a mentor, a friend, a rival —
whose storylines bend around what you actually wrote.

The fifteen-year frame is the point. Individual months are small; the shape only
appears in retrospect, which is also true of the thing being simulated.

---

## Design Process

Every system was written as a design document before it was written as code, and
the architecture was settled before the first system was implemented. The order
was: concept → system decomposition → per-system design docs → architecture
decisions → implementation.

**Why work this way on a solo project**, where there is no one to hand the
document to:

- **The document is the thinking.** Writing "how does reputation decay interact
  with the awards season" in prose exposes contradictions that code hides behind
  plausible-looking arithmetic. A meaningful share of this project's design
  changes were made at document stage, before there was any code to throw away.
- **It gives an AI agent something to be checked against.** This matters more
  than it does with human collaborators, and is discussed below. An agent asked
  to "implement the ending system" will produce something confident and wrong.
  An agent asked to implement a documented rule can be checked against that rule.
- **It makes staleness detectable.** The project treats a design document with an
  out-of-date number as a defect, not as an inconvenience — there is a standing
  rule that any change touching behavior updates its document in the same commit,
  and several automated checks enforce specific cases of it.

The design documents themselves are not published here — they are the game's
content in an early form. What is published is the process.

**Where it broke down, honestly:** documents go stale in ways discipline alone
does not catch, and stale documents are worse than absent ones because they are
believed. Several of the harder bugs in this project trace to a document
asserting a design premise that the code had quietly moved away from months
earlier. The fix was not more discipline; it was moving load-bearing claims out
of prose and into checks that fail loudly.

---

## Technical Architecture

**Four autoloaded singletons** hold everything global: an event bus, a data
manager, a theme manager, and the game state. Nothing else is global.

**Twenty-one gameplay systems** are plain nodes, each in its own file, each
independently constructible and testable. Systems never call each other
directly — all cross-system communication goes through signals on the event bus.
The month loop is the only clock: it advances time and emits, and everything
else reacts. This means a system can be exercised in isolation by emitting the
signals it listens for, which is what the test harness does.

**The UI is a separate layer with its own orchestration.** Screen controllers
never reach into gameplay systems; they listen to the same event bus and talk
upward to a single orchestrating view through signals. As the orchestrator grew,
cohesive slices were extracted into presenters — one for modal interactions, one
for the web-serial screens, one for the guided tutorial rail, one for save and
session handling. The orchestrator has a line-count ceiling enforced in CI —
a crude tool, but one that has repeatedly forced an extraction that would
otherwise have been deferred indefinitely.

**Content is data, not code.** All game content lives in JSON: the three
creation axes and their compatibility matrix, 119 work traits across four rarity
tiers, events split by class, 24 publishing channels, 21 endings, 51
achievements, and the three NPC storylines. Designers — which is to say, me
wearing a different hat — add content by editing data files, never scripts.

A small shared **condition evaluator** makes this work. Designers write trigger
conditions as strings in the data files (`works_published>=2`,
`active_track_web>=1 and signed_ever>=1`), and a single resolver interprets them
against game state. One vocabulary, one implementation, one lint gate that
rejects unknown terms at build time — so a typo in a data file fails the build
instead of silently making a story beat unreachable.

**Localization is structural, not a late pass.** Every player-visible string
goes through a translation lookup from the first line of code. The CSV is the
only source of truth in version control; the compiled translation binaries are
build products, regenerated on every import and never committed. That single
decision eliminated an entire class of bug — "edited the strings, forgot to
recompile, shipped the old text" — by making the stale artifact impossible to
commit.

**Determinism where it matters.** Anything that needs to be reproducible —
market era rotation, monthly rankings, contest opponents — is a pure function of
`(seed, month)` rather than accumulated random state. Reopening a window shows
the same numbers; a saved game reloads into the same world; and a bug that
appears at month 94 can be reproduced by replaying from month 1 with the same
seed.

### Testing

This is the part I would most want to talk about in an interview.

A single command runs the full gate in about eighteen seconds and returns a
meaningful exit code. It performs a headless logic sweep, then four
progressively nastier UI gates:

- **Boot gate** — the main scene loads without error.
- **Fit gate** — every window and multi-step flow is measured at 720p, 1080p and
  ultrawide. Windows may not exceed the screen; the primary action button may not
  drift between steps of a flow; the bottom dock must stay full-width. Any new
  window has to be added to this probe to be merged.
- **Modal-layer gate** — the game has two independent overlay stacks, and they
  must be mutually exclusive. When they are not, the player gets a window they
  cannot dismiss. The probe asserts the invariant directly rather than hoping
  someone notices.
- **Focus gate** — two focus-containment nets in the same frame can fight over
  deferred focus reclaims, flood the call queue, and crash the process. The probe
  reproduces the contention deliberately and turns a real crash into an
  assertable failure.

The last two exist because both bugs actually shipped, once each. Turning a
crash into a red test is the most durable form of a bug fix.

Alongside the harness, a **scripted player** plays complete fifteen-year runs
headless from a fixed seed. Running several hundred of them yields an ending
distribution, which is how balance changes are evaluated: a tuning change is
accepted or rejected on whether the distribution moved outside its confidence
band, not on whether the change felt right. That method also taught me its own
limitation — in a system with this much randomness, an A/B comparison of a single
knob is meaningless, because removing one dice roll re-sequences every subsequent
roll and the two runs diverge completely. Measuring the window in which a
mechanism is actually active turned out to be the only honest approach.

A parallel set of **lint gates** covers content rather than behavior: orphaned
translation keys, generated documents that have fallen out of sync with their
sources, placeholder tokens used on beats that cannot supply them, banned
glyphs, missing translations, and prose fields containing author to-do notes
that would otherwise render to the player. Content bugs in a data-driven game
are as shippable as code bugs and considerably easier to overlook.

**CI** re-runs the whole thing on Linux for every push and pull request. It
exists to catch the class of failure a single development machine structurally
cannot: case-sensitive path breaks, assets that were never actually committed,
and build products that only regenerate correctly from a clean clone.

---

## AI-Assisted Development

This project was built with Claude Code as the primary development agent. That
is the honest description: not "AI-assisted autocomplete," but a workflow where
an agent does most of the typing and the human does the specifying, the
adjudicating, and the verifying. It is worth describing precisely, including
where it does not work.

### Workflow

The loop that settled out, after several that did not:

1. **Play the build and write down what is wrong** — in plain language, as a
   player, not as a bug report. Fifty-odd rounds of this exist.
2. **Turn each complaint into a claim that can be false.** "The NPC feels
   distant" becomes "does any code path actually raise this relationship above
   40, and if not, why not." This step is the entire job; an agent aimed at a
   feeling will produce a plausible-sounding change that fixes nothing.
3. **Dispatch parallel agents for breadth**, one per subsystem, with explicit
   instructions to cite file and line for every assertion and to say "not found"
   rather than guess. Their findings are treated as leads, not conclusions.
4. **Verify the leads personally** before acting on any of them.
5. **Fix, then pin.** A fix that is not accompanied by a check which fails
   without it is not finished.
6. **Update the documents in the same commit.**

Two supporting pieces make this practical. A **persistent memory** of
project-specific rules and past mistakes is committed alongside the code, so
lessons survive across sessions and across models. And a **live inspection
bridge** into the running game lets the agent read actual runtime state and take
real screenshots, rather than reasoning about what the code probably does — which
is the difference between debugging and guessing.

### Across model generations

The workflow was rebuilt roughly three times as the models changed.

**Opus 4.6** was the model this project was designed around. It was strong at
localized, well-specified work: implement this documented system, refactor this
file, write these thirty content entries to this schema. It needed the
specification to be tight, and it lost the thread across long multi-file
changes, so the work had to be cut into pieces small enough to hold. The heavy
documentation-first process in this repository is partly a response to that.

**Opus 5** changed what a single task could be. Work that previously had to be
decomposed into six specified steps could be handed over as one objective with
its constraints, and it would hold the whole shape — including the parts of the
codebase it had to go read to do the job. The practical consequence was that the
bottleneck moved from *specifying* to *verifying*: it now produces more
plausible work per unit time than a human can carefully check, which makes the
automated gates load-bearing rather than nice to have. It is also markedly
better at the thing that matters most here — noticing that the premise of the
request is wrong and saying so before implementing it.

**Sonnet 5** is what most of the mechanical volume actually runs on: batch data
edits, translation passes, applying a decided change across forty files,
first-pass audits. On well-bounded work its output is close to indistinguishable
from the larger model at a fraction of the cost, and the cost difference is what
makes running five agents in parallel a normal thing to do rather than an
indulgence. It is less good at knowing when to stop and ask.

**Fable 5** earned a specific role: prose. This game carries about 54,000
characters of Chinese narrative text — NPC dialogue, inner monologue, event
descriptions, ending epilogues — each of which also exists in English and
Japanese, and the failure mode of AI-written fiction is very particular: technically fluent, tonally
generic, over-explained, reaching for a summary sentence the scene did not need.
Fable 5 is noticeably better at restraint and at holding a consistent voice
across hundreds of separate short pieces, which is the actual difficulty at this
scale. Its output still gets edited. It gets edited less.

Running a mixed fleet is not free. The models disagree, and a change made by one
gets reviewed by another that reaches a different conclusion with equal
confidence. The resolution is always the same: whichever position can be
attached to a runnable check wins, and if neither can, the disagreement is a
signal that the question was underspecified.

### Tools that earned their place

- **A studio-role agent pack** — dedicated designer, QA, narrative, UX, and
  Godot-specialist agents, each with its own persistent memory. The value is not
  role-play; it is that a reviewer with a narrow brief and its own accumulated
  context finds different problems than a general-purpose one, and their memories
  do not contaminate each other.
- **A structured brainstorm-then-plan skill** that refuses to write code until a
  design has been stated and approved. It is deliberately obstructive and it has
  paid for itself repeatedly, because the failure it prevents — building the
  wrong thing efficiently — is the expensive one.
- **A systematic-debugging skill** enforcing reproduce-before-fix. Most of my
  own worst debugging sessions on this project began with skipping that step.
- **An anti-over-engineering skill** that biases every proposal toward the
  smallest change that works. The default failure mode of a capable coding agent
  is not writing bad code; it is writing three times as much correct code as the
  problem required, complete with configuration flags nobody will ever set.
- **The Godot inspection bridge** described above.
- **A documentation-reconciliation skill** that audits docs and memory against
  the actual code at session end.

### Where it is reliable, and where it is confidently wrong

**Reliable:** mechanical transformations across many files; generating content
that conforms to a schema; writing test probes and lint rules; breadth-first
audits of an unfamiliar area; translation with a glossary; anything where
correctness is checkable by a program.

**Confidently wrong — real examples from this repository:**

- **Reasoning from an assumed structure instead of the actual one.** A script
  written to synchronize a field across sibling data files assumed a consistent
  key order. One file ordered its keys differently, so every entry in it received
  the *previous* entry's values. The code was clean, the logic was sound, and the
  output was corrupt. It was caught only because the script reported 202 changes
  against a drift I had already measured at 46 — the count disagreed, so the work
  was reverted.
- **Writing a check that is subtly wrong in the permissive direction.** A new
  lint rule intended to catch author to-do notes left in player-facing text
  flagged a genuine line of dialogue on its first run, because the pattern was
  too loose. A gate that produces false positives gets disabled, and a disabled
  gate is worse than no gate.
- **Treating stale documentation as current fact.** Agents believe the
  repository's own documents. When a design note asserted that a subsystem
  behaved one way and the code had done something else for two months, every
  decision downstream of that note inherited the error.
- **Reporting a gate as green when it was not.** A shell pipeline swallowed the
  real exit code; the harness had failed and the summary said it passed. Twice.
- **Silencing a check instead of satisfying it.** A gate correctly flagged six
  items; all six were added to a whitelist in one pass on the grounds that they
  had been reviewed. Five had been. Auditing that whitelist months later found
  the sixth, which was exactly the bug the gate was written to catch.

The pattern connecting all five is that the failure is never a syntax error or
an obviously bad idea. It is always something that looks right, is internally
consistent, and is wrong about a fact nobody re-checked. That is why the working
rules in this project are what they are: a fix is not done until a number that
was wrong is observably right; a new gate must be deliberately broken and seen to
fail before its green is believed; and when an instrument reports something
implausible, the instrument is suspected before the world is.

None of this is an argument against working this way. This project is
substantially larger and better tested than anything I would have built by hand
in two months. But the leverage is entirely on the production side, and it moves
the human's job upstream, into deciding what is true and building the machinery
that keeps proving it.

---

## Local LLM Pipeline

Early in the project I ran translation locally rather than through an API:
**Qwen3 4B on Ollama**, on a MacBook M4, translating Chinese to English — book
titles from the generated title pools, and passages of story text.

**Why local, and why such a small model.** Two reasons, one practical and one
curious. The practical one was budget: token allowance was tight at the start,
and translating a large volume of short strings is exactly the kind of
high-count, low-difficulty work that would have consumed a disproportionate
share of it. Spending that allowance on architecture and system implementation
was clearly the better trade. The curious one was that I wanted to find out how
a 4-billion-parameter model actually performs at translation, rather than
assuming it could not.

**How it went.** Better than expected. For self-contained strings — a book
title, a short line — the output was usable, and running locally meant zero
marginal cost per call, no rate limits, and no round trip.

**Why I stopped.** Context length. Translating this game's text well is not a
string-by-string problem: it needs the glossary, the surrounding scene, and the
established voice of the character speaking, all in context at once, or the same
term comes out three different ways in three different places and the register
drifts between adjacent lines. A 4B model with a short context window cannot hold
that, and stitching it together with retrieval was going to cost more engineering
than the token savings were worth. The project moved fully to Claude for
translation, and the trilingual string table is now maintained as a single
consistent artifact with automated checks for missing and drifted entries.

The honest summary is that the local model was the right call for the constraint
I had at the time, and the wrong call once the constraint changed — and the
limiting factor was context, not raw quality.

---

## Source Code

**The source code and art assets for this project are not public**, and this
repository contains none of them. It is a write-up only.

Read-only access to the private source repository can be provided on request for
hiring purposes. If you would like a walkthrough of any specific system — the
test harness, the data-driven content pipeline, the localization setup, or the
NPC storyline engine — I am happy to go through it.

**Bowen Li** · [github.com/Bli18](https://github.com/Bli18)
