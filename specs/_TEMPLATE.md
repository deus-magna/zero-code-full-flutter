# SPEC-XX — [Short feature title]

> Copy this file to `specs/SPEC-XX.md`. Delete sections that truly do not apply, but never
> leave "Acceptance criteria" or "Out of scope" empty. If something is ambiguous, write it under
> "Open questions" — the orchestrator will resolve it with the human before planning.

## 0. User directives and reference UI

**User directives** — copy here, verbatim, everything the user said about this feature,
especially anything like "use / reuse / make it like X". These are binding: the plan review and
the Reviewer check them. "None." only if the user truly gave none.

- "..."

**Reference UI and reuse decisions** — one row per screen, widget, dialog, entity or l10n set that
exists elsewhere and is similar. The orchestrator asks the user for the decision *before* filling
this in (see "Reuse & shared UI" in AGENTS.md); never decide it silently. "None." if nothing similar
exists.

| Piece | Reference file(s) | Decision (reuse as is / extract generic to `features/common` / copy) | Original feature migrated? | Approved by user |
|---|---|---|---|---|
| ... | `lib/features/.../x.dart` | ... | yes / no (future spec) | yes |

## 1. Motivation

[What problem does this solve for the user? Why now? One short paragraph.]

## 2. Behavior

[What the user sees and can do. Tables for conditional behavior. Include empty, loading and
error states, and any difference between user types (e.g. free vs. paid) if relevant.]

| Condition | Result |
|---|---|
| ... | ... |

## 3. Architecture

[Which feature(s) under `lib/features/` are touched or created. Whether it extends an existing
Bloc or adds a new one. A small diagram of the data flow helps.]

```
View → Event → Bloc → UseCase → Repository → DataSource
```

## 4. Domain (entities, repository contracts, use cases)

[Entities with fields and types. Repository contract methods. One use case per action, with its
inputs/outputs. Any business logic belongs here.]

## 5. Data (models, datasources, persistence)

[Models and their serialization. Where the data lives (endpoint, table, collection, local
storage) and its shape.]

**Persistence & security** — list every rule a write must satisfy (required fields such as an
owner id or timestamps, server-side validation, permissions), as described in "Project specifics"
in AGENTS.md. State which existing model this one copies: [model name]. "Not applicable." if
nothing is persisted.

**Compatibility with existing data** — for every persisted shape this spec touches:
- New fields are nullable/defaulted on read: [yes/no, how]
- Data saved by older versions keeps working: [how]

## 6. Presentation (bloc events/states, views, widgets)

[Bloc events and states. Screens and widgets, one class per file. l10n keys needed (reuse existing
keys when the meaning matches; list the reused keys and only the genuinely new ones).
Error/validation messages.]

**UI values to match** — when the section 0 table references another screen, list the concrete
values to copy from its source (spacing between sections, field order, labels/titles, button
texts, number/date format, max lengths, number of lines). The Reviewer compares against these.

## 7. Navigation

[Routes added or changed. Existing paths must not change. "None." if not applicable.]

## 8. Feature flags / monetization (optional)

[Is anything gated (paid tier, feature flag, role)? Which existing widgets or Blocs handle the
check? "Not applicable." if not.]

## 9. Backward compatibility & risks

[What could break released behavior. Entry points or flavors affected. Anything needing manual
testing on a device.]

## 10. Acceptance criteria

[Verifiable, numbered. The Reviewer checks these one by one. Include: one criterion per user
directive from section 0, one criterion for UI parity with each reference (with the concrete
values), and one for the persistence & security rules of section 5 if anything is persisted.]

1. ...
2. ...

## 11. Out of scope

[What the Implementer must NOT do in this spec. These count as criteria: violating one is a failure.]

- ...

## 12. Open questions

[Anything unresolved. Must be empty before the plan is written. "None." when clear.]
