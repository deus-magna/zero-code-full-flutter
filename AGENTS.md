# AGENTS.md

This file gives strict guidance to AI coding agents (Claude Code, opencode) working in this Flutter
project. Follow these rules exactly when generating or modifying code.

`CLAUDE.md` only imports this file, so this is the single source of truth.

> **Using this kit in your own project?** Everything below is generic. Fill in the
> [Project specifics](#project-specifics) section with your backend, navigation and anything else
> that is particular to your app, and adjust the architecture rules if your team uses a different
> one. The [Spec-Driven Development](#spec-driven-development-sdd) part works with any architecture.

---

## Commands

```bash
flutter pub get
flutter run
flutter test
flutter test test/path/to/test.dart
flutter analyze
flutter build apk
flutter build ios
```

If your app uses flavors or several entry points, list the exact commands here so agents run the
right ones.

## Architecture Overview

This project uses Clean Architecture with BLoC (events + states) as the only state management
solution.

The architecture is strict and enforced. All features must follow the same structure and data
flow.

Packages this architecture relies on (add them the first time a feature needs them):
`flutter_bloc`, `equatable`, `get_it`, and for tests `bloc_test` and `mocktail`.

---

## Folder Structure

All features live under:

```
lib/features/<feature_name>/
```

Each feature MUST follow this structure:

```
data/
  datasources/
  models/
  repositories/

domain/
  entities/
  repositories/
  usecases/

presentation/
  bloc/
  view/
  widgets/

dependencies/
  <feature_name>_dependencies.dart
```

A feature with several distinct flows nests sub-features one level deeper, each with its own full
`data/domain/presentation/dependencies` set (e.g. `lib/features/orders/checkout/` and
`lib/features/orders/history/`).

Code shared by several features lives in `lib/features/common/` (see "Reuse & shared UI").

## Data Flow (MANDATORY)

Every feature MUST follow this exact flow:

```
View → Event → Bloc → UseCase → Repository → DataSource
                                           ↓
View ← State ← Bloc ← UseCase ← Repository ← DataSource
```

### Rules:

- A View NEVER talks directly to a repository or datasource
- A Bloc NEVER skips a UseCase
- A UseCase ALWAYS uses a Repository interface (from domain)
- A Repository implementation uses DataSources
- Data always flows back through the same chain

## Layer Responsibilities

### Presentation Layer
- Contains UI (views, widgets) and BLoC
- Uses events to trigger logic
- Reacts to states emitted by Bloc
- No business logic allowed
- Form validation and input formatting belong here; the BLoC and UseCases own any business
  calculation
- Keep it modular — one widget class per file, split by responsibility:
  - `presentation/view/`: the Screen for a route. It wires up `BlocProvider`/DI and owns the
    screen's overall widget tree.
  - `presentation/widgets/`: every other widget substantial enough to matter (a form field, a
    card, a result summary) gets its own file.
  - Don't stack multiple widget classes in one file, and don't leave a private widget (e.g.
    `_FooCard`) inline in a View file once it's more than a few lines — give it its own file in
    `widgets/` (as a public class, since a leading-underscore name can't be imported across
    files) even if it's only used in one place. This keeps files short and easy to read.

### Domain Layer
- Pure Dart (no Flutter, no backend SDKs)
- Contains:
  - Entities (Equatable)
  - Repository contracts (abstract classes)
  - UseCases (single responsibility actions, including business calculations)

### Data Layer
- Implements domain repositories
- Contains:
  - Models (DTOs, mappers)
  - DataSources — the only place that talks to the outside world (HTTP API, backend SDK, local
    storage). See [Project specifics](#project-specifics) for this project's backend.
  - Repository implementations
- DataSources can be remote or local (e.g. `SharedPreferences` for cached settings) — both are
  valid, just keep them behind the Repository interface.
- Every entity that needs to reach a DataSource (read or written) MUST have a corresponding
  Model in `data/models/` with `toJson`/`fromJson` (or `fromMap`/`toMap`) as applicable — a
  write-only entity only needs the serializing direction, a read-only one only the parsing
  direction. The Repository implementation converts entity → Model before calling the
  DataSource, and Model → entity for what it returns. Do NOT write one-off cast/mapping functions
  instead of a proper Model.

## BLoC Rules (VERY IMPORTANT)

- Every feature MUST have:
  - Bloc
  - Event
  - State

- Naming convention:
  - <Feature>Bloc
  - <Feature>Event
  - <Feature>State

- A Bloc:
  - Receives Events
  - Calls UseCases
  - Emits States

## When Creating a New Feature

Agents MUST always generate ALL required layers:

- Entity
- Repository contract
- UseCase(s)
- DataSource(s)
- Model(s)
- Repository implementation
- Bloc (events + states)
- View
- Widgets (if needed)
- `dependencies/<feature_name>_dependencies.dart` exposing an `init<Feature>Dependencies()`
  function that registers the BLoC, UseCases, Repository, and DataSources with `sl` (GetIt) —
  then call it from `lib/features/common/dependencies/injection_container.dart`'s `init()`
  (create that file with the first feature if it does not exist yet)

Never create partial implementations.

## When Updating an Existing Feature
- Reuse existing structure
- Extend existing BLoCs instead of duplicating
- Avoid creating duplicate UseCases or repositories
- Maintain backward compatibility unless explicitly told otherwise
- If the app already has users, treat persisted data as a contract: when a change alters the
  shape of stored data, make it additive/backward-compatible (e.g. nullable new fields, defaulting
  on read) rather than a breaking rewrite

## Reuse & shared UI (MANDATORY)

Before writing or planning any screen, widget, dialog, entity or l10n string, check whether
another feature already has an equivalent. Duplicating existing UI is a defect, not a shortcut.

- **Ask before deciding.** When the user mentions an existing screen, widget, dialog, entity or
  feature as a reference ("make it like X", "use the UI of X"), the orchestrator MUST ask the user
  how to reuse it *before* writing the spec. Never decide silently. Options:
  1. **Reuse as is** — only if it has no feature-specific dependencies (another feature's Bloc,
     entities or routes).
  2. **Extract a generic, presentational widget into `lib/features/common/presentation/widgets/`**
     and use it from the new feature. Migrating the original feature to the shared widget is a
     separate decision: the original may already be released, so it needs the user's approval.
  3. **Copy and adapt.**
  Recommend option 2 by default and flag the risk of touching the original.
- **Coupling rule.** A feature never imports another feature's presentation layer, Bloc or
  entities. Anything shared goes in `lib/features/common/`. `common` should not depend on a
  feature; if a shared widget needs something that lives in a feature, say so in the plan and
  propose moving it instead of silently importing it.
- **The user's words are binding.** Copy every directive the user gives about the feature
  (especially "use / reuse / like X") verbatim into the spec under "User directives". The plan
  review and the Reviewer check them.
- **Reuse l10n keys.** Use an existing key when its meaning matches. Create a new key only for a
  new meaning. Never change the value of an existing key other screens use; add a new key instead.
- **Match references with concrete values.** When a spec says "like X", copy the real values from
  X's source (spacing, field order, labels, formats, button texts). Do not approximate. Read X's
  model and datasource too, not only its entity.
- **Analogous entities are evidence.** If another feature already models the same concept, model
  the new one after it or ask why not.

## Critical Rules
- Do NOT bypass layers
- Do NOT mix responsibilities between layers
- Do NOT put business logic in UI
- Do NOT access the backend, network or storage outside DataSources
- Do NOT create tightly coupled code between features; shared code goes in `features/common` (see
  "Reuse & shared UI")
- Do NOT let a domain Entity reach a DataSource. Entities live in `domain`/`presentation`; the
  `data` layer uses Models (see Data Layer). A Repository implementation converts entity → Model
  before calling a DataSource, never the entity itself
- Do NOT change the shape of persisted data in a breaking way once the app has real users

## Agent Behavior Expectations

When working in this repo, agents MUST:

- Respect Clean Architecture strictly
- Always implement full data flow
- Generate complete feature scaffolding when needed
- Follow naming conventions consistently
- Prefer maintainability over shortcuts
- Favor safe, backward-compatible changes

## Thinking Process (IMPORTANT)

Before writing code, agents must:

- Identify the feature and its boundaries
- Determine required entities and use cases
- Design the data flow
- Then generate code following all layers
- Do NOT jump directly into UI or implementation.

---

## Project specifics

> Fill in this section for your app. Agents read it before planning and implementing. Delete the
> items that do not apply. Being concrete here (exact file paths, rules, names) prevents bugs that
> no analyzer catches.

- **Backend & data:** which backend the DataSources talk to (REST API, Firebase, Supabase, local
  only…), where collection/table/endpoint names are defined, and the conventions every model must
  follow (e.g. required fields such as an owner id or timestamps).
- **Security rules:** any server-side rule a write must satisfy. If a write fails at runtime when a
  field is missing, write it here explicitly — the analyzer will not catch it.
- **Navigation:** router and where routes are defined; which paths are public or deep-linked and
  must never change.
- **App initialization:** where DI, SDKs and configuration are set up, and which entry points or
  flavors exist.
- **Monetization / feature flags:** how paid or gated features are checked, and which existing
  widgets/Blocs to reuse instead of inventing a new check.

---

## Language

Everything in the repo's collaboration surface is written in **English**: code comments, commit
messages, PR titles and descriptions, code review comments and GitHub issues. Specs and plans keep
the language the human uses with the orchestrator. (Change this rule if your team works in
another language.)

---

## Workflow tools

### Crit — plan review

[Crit](https://github.com/tomasz-tomczyk/crit) is the feedback channel between the orchestrator and
the human. It is **mandatory** for reviewing the plan before implementing any spec.

1. The Planner writes the plan to the path the orchestrator gives it (see "Plan files and iterations")
2. The orchestrator calls `crit <that plan path>`
3. **The orchestrator stops completely.** It implements nothing and does not advance.
4. The human reviews the plan in Crit, leaves inline comments and clicks "Finish Review"
5. Crit hands the structured feedback back to the orchestrator
6. If there are comments: invoke `planner` again with the feedback, regenerate the plan, repeat from step 1
7. If there are no comments or the human approves: invoke `implementer`

**Critical rule: zero code before Crit approval.** Not a single line, not even scaffolding.

### Engram — persistent memory across specs

[Engram](https://github.com/Gentleman-Programming/engram) keeps context between specs so the
orchestrator does not start from scratch each time.

| Moment | Tool | What to save / search |
|---|---|---|
| Start of each spec | `mem_search` | Context from earlier specs (e.g. "auth bloc", "cart repository", "pagination") |
| Start of each spec | `mem_context` | Recover the previous session summary if there was a compaction |
| After completing a spec | `mem_save` | Technical decisions, discovered patterns, solved problems |
| After completing a spec | `mem_session_summary` | Structured summary (Goal / Discoveries / Accomplished / Files) |
| After any compaction | `mem_context` | **Mandatory** — recover state before continuing |

Worth a `mem_save` in this project:
- Architecture decisions made during implementation
- Persisted-data shape decisions and how backward compatibility was kept
- DI / BLoC patterns that worked (or didn't) in a specific way
- Any deviation from the spec and why
- Conventions the Implementer applied that were not explicit in the spec

Context recovery, in 3 layers (minimum tokens): `mem_search` → `mem_timeline observation_id=N` →
`mem_get_observation id=N`. Search first, go deeper only where relevant.

---

## Spec-Driven Development (SDD)

Every feature or non-trivial change goes through a spec. Specs live in `specs/SPEC-XX.md`
(start from `specs/_TEMPLATE.md`); plans live under `specs/plans/` (see "Plan files and iterations").

```
SPEC → Engram (search context) → Explorer → Planner → Crit (approve plan)
     → Implementer → Reviewer → test-writer → code-reviewer → Engram (save)
```

### Plan files and iterations

A spec can need more than one plan: an amendment after the first plan was implemented, new scope
after a review, a rework requested by the user. Keep them ordered so it is always clear which came
first:

- **First plan:** `specs/plans/SPEC-XX-plan.md` (iteration 1, id `SPEC-XX`).
- **From the second iteration on:** create the folder `specs/plans/SPEC-XX-plan/`, move the first
  plan into it (keep its name `SPEC-XX-plan.md`, use `git mv` if tracked) and name each new plan
  `SPEC-XX.NN-<slug>-plan.md`, where `NN` is the iteration number, two digits, starting at `02`
  (e.g. `SPEC-02-plan/SPEC-02-plan.md`, `SPEC-02-plan/SPEC-02.02-ui-plan.md`,
  `SPEC-02-plan/SPEC-02.03-dialogs-plan.md`). Alphabetical order is then chronological (`-` sorts
  before `.`). `SPEC-XX.NN` is a plan id, not a new spec: the spec stays `specs/SPEC-XX.md`.
- **Header:** every plan starts with a line right after its title:
  `> Plan ID: SPEC-XX.NN · Iteration: N · Amends: <previous plan id> (`file`)`
  (`Amends: none` for iteration 1).
- **When a new iteration is created:** only when the previous plan was already implemented (fully
  or partly) or is superseded by new scope. Crit review rounds on the same plan edit that file in
  place; they are not new iterations.
- **The orchestrator decides the path** (and creates the folder / moves the earlier plan when
  needed) and passes it to the Planner; the Planner never picks it. Update the spec when an
  iteration changes it, so the spec stays the source of truth.

### Step by step

**Step 0 — Spec.** The human and the orchestrator turn the idea into `specs/SPEC-XX.md`.
If the idea is ambiguous, the orchestrator asks the human *before* anything else. If the idea
references an existing UI/feature, the reuse decision is asked here (see "Reuse & shared UI") and
the user's directives are copied verbatim into the spec. The spec must have acceptance criteria,
an "Out of scope" section and no open questions.

**Step 1 — Recover context (Engram):** `mem_search <key terms>` and `mem_context`. Also check the
backlog: `gh issue list --label sdd-followup` for open follow-ups related to the spec, and mention
them to the human.

**Step 2 — Explore (`explorer`):** invoke with the spec. Produces the current-code context report,
including the **reusable inventory** of other features (widgets, dialogs, entities, l10n keys).

**Step 3 — Plan (`planner`):** invoke with the spec + Explorer report. The Planner writes the plan
file (path given by the orchestrator), including the mandatory "Compatibility & production impact"
section and a "Reused vs new" section that justifies every new widget, dialog or l10n key.

**Step 4 — Plan review (Crit) — HARD BLOCK:** `crit <plan path>`, then **stop all activity** until
the human finishes the review. Feedback → back to Step 3. Approved → Step 5.

**Step 5 — Implement (`implementer`):** one invocation per task, following the approved plan.
Do not start the next task until the current one passes `flutter analyze`. After the last task the
Implementer saves its notes to Engram.

**Step 6 — Review (`reviewer`):** invoke with the spec + implemented files. It also checks the
reuse decisions, UI parity against the referenced screens and the rules in
[Project specifics](#project-specifics). If criteria fail, go back to Step 5 for the failed tasks
only, then Reviewer again.

**Step 7 — Tests (`test-writer`):** once the Reviewer reports `✅ SPEC COMPLETE`, invoke it on the
new/changed feature, in "step by step" mode by default. It follows its own hard rules (never fix
production code to make a test pass; no trivial tests).

**Step 8 — Code review (`code-reviewer`):** once the tests pass, invoke it on the branch diff
against `main`, before opening the PR. It is a tech-lead style review (bugs, production risks,
performance, maintainability, test quality), independent of the spec's criteria, and read-only.
Blocking findings go back to Step 5 for those fixes only. Suggestions and nits are the user's call.
It can also be invoked on demand for any diff or PR, outside the SDD flow.

**Step 8b — Follow-up triage (human decides):** after the code review, the orchestrator gathers
every item from the "Follow-ups" sections of the agents' reports (Planner, Implementer notes,
Reviewer, code-reviewer) and from decisions the human deferred ("for another spec"). Before asking,
it searches open issues (`gh issue list --state open --search "<terms>"`) to detect duplicates. It
then shows the list in ONE message and the human picks one outcome per item: **create issue**,
**already exists** (link the number) or **discard**. Issues are created only after that approval,
with the template in `.github/ISSUE_TEMPLATE/followup.md`, the `sdd-followup` label plus `bug` or
`enhancement`. Anything that blocks the plan stays an "Open question", not a follow-up.

**Every time the user asks to create a PR**, the orchestrator, right after `gh pr create`:
1. Invokes `code-reviewer` on the PR (`gh pr diff <n>`), locally on the checked-out branch.
2. Has it publish the findings to the PR as comments, for traceability. This is a standing
   authorization from the user, limited to comments: never a formal approve or request-changes,
   never pushing code, never editing the PR.
3. Reports the verdict and the comment links to the user. Blocking findings are not fixed
   automatically; the user decides.

**Step 9 — Persist memory (Engram):** `mem_save` and `mem_session_summary`.

### Agents and models

Claude Code is the primary tool. opencode is an optional alternative with the same agents. In
opencode, the orchestrator is the primary agent `sdd` (`.opencode/agents/sdd.md`); in Claude Code
the main session plays that role.

| Agent | Claude Code model | opencode model | When |
|---|---|---|---|
| `explorer` | haiku | your choice | Step 2: read and map existing code |
| `planner` | sonnet | your choice | Step 3: turn the spec into a task plan |
| `implementer` | sonnet | your choice | Step 5: write code task by task |
| `reviewer` | sonnet | your choice | Step 6: validate against acceptance criteria |
| `test-writer` | (session default) | your choice | Step 7: unit tests for domain, data and bloc layers |
| `code-reviewer` | sonnet | your choice | Step 8: tech-lead style review of the branch diff before the PR |

Models are set in each agent's frontmatter. Change them to fit your budget.

**The agents are intentionally duplicated** — `.claude/agents/` (Claude Code) and
`.opencode/agents/` (opencode) — because each tool needs its own frontmatter. The body of each
agent is identical in both folders. **When you edit an agent, apply the same change to both copies.**

### Orchestrator rules

- **Do not skip steps.** Order: Engram → Explorer → Planner → Crit → Implementer → Reviewer → Tests → Code review → Engram.
- **Crit is a real block.** No code before the plan is approved.
- **One agent at a time.** Wait for the full response before invoking the next.
- **One Implementer per task.** Never hand the whole plan to the Implementer at once.
- **If the Reviewer reports failures**, go back to the Implementer only for the failed tasks, then Reviewer again.
- **Do not mix specs.** Complete and persist in Engram before accepting the next one.
- **One branch per spec, created by the orchestrator.** Before Step 2, check the current branch. On
  `main`, create `feature/SPEC-XX-<feature-name>`. On a feature branch for a different spec, stop
  and ask the human. The Planner never runs git (it has no Bash); it only works on the branch the
  orchestrator prepared.
- **On spec ambiguity**, ask the human before planning.
- **After any compaction**, call `mem_context` immediately before continuing.
- **Users first.** A plan or implementation that breaks persisted data, a public route or an
  existing Bloc contract is rejected, regardless of acceptance criteria.
- **Ask before deciding reuse.** If the user references an existing UI, entity or feature, ask how
  to reuse it (see "Reuse & shared UI") before writing the spec. Do not pick for them.
- **Copy the user's directives into the spec** ("User directives" section) and verify them at plan
  review and in the Reviewer.
- **Crit `approved: false` is not approval.** If Crit finishes with `approved: false` and the only
  open comments were already answered (carried forward), ask the user whether to approve; never
  assume it.
- **Small fixes are exempt.** Typos, copy changes, a one-line bugfix with an obvious cause may skip
  the full flow; say so explicitly to the human instead of silently skipping.
