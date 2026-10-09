# Domain 3 — Claude Code Configuration & Workflows

**Blueprint weight: 20% (~12 of 60 questions) · only 30 questions in the bank, ~18 distinct concepts**

> ⚠️ **This is your biggest coverage gap — and it is worse than the count suggests.** D3 carries 20% of the exam but 13% of the bank, and it has the **highest duplication rate** of any domain: almost every numbered question has a screenshot twin testing the identical scenario. Thirty questions reduce to roughly eighteen concepts. Study this domain from the official lessons, not from practice questions.

> **The one sentence that governs this domain:** choose the mechanism by **how strongly the rule must hold** and **how widely it applies**.

```
                 How strongly must it hold?
                 ─────────────────────────►
   Advisory                              Absolute
   ┌──────────────┬───────────────┬──────────────────┐
   │ CLAUDE.md    │ .claude/rules/│ settings.json    │
   │ (always on)  │ (path-scoped) │ permissions      │
   ├──────────────┼───────────────┼──────────────────┤
   │ Skills /     │ @path lines   │ Hooks            │
   │ slash cmds   │ (modular,     │ (deterministic)  │
   │ (on demand)  │  always load) │                  │
   └──────────────┴───────────────┴──────────────────┘
```

| If the rule is... | Use |
|---|---|
| General background guidance, always relevant | `CLAUDE.md` |
| Only relevant when certain files are touched | `.claude/rules/` with `paths:` frontmatter |
| A modular chunk of an otherwise relevant doc | An `@path/to/file.md` line |
| A reusable workflow invoked when needed | Skill or slash command |
| Non-negotiable | `settings.json` permissions, or a hook |

---

## 3.1 CLAUDE.md hierarchy, scoping, and modular organisation

### What CLAUDE.md actually is

`CLAUDE.md` files are **loaded into context as guidance**. They are not a strict override chain, not a config schema, and not an enforcement mechanism. Claude Code discovers them at several levels and the applicable content is **concatenated**:

| Location | Scope | Shared with the team? |
|---|---|---|
| `~/.claude/CLAUDE.md` | User — every project on your machine | ❌ Never reaches a teammate |
| `.claude/CLAUDE.md` **or** root `CLAUDE.md` | Project | ✅ If committed |
| `<subdirectory>/CLAUDE.md` | That part of the tree | ✅ If committed |
| `CLAUDE.local.md` (any level) | Personal, for **this repo** | ❌ Gitignored by convention |

Both `.claude/CLAUDE.md` and a root `CLAUDE.md` are valid project-level locations — the exam may show either path.

### Loading order is not precedence

Anthropic's memory documentation is explicit: *"All discovered files are concatenated into context rather than overriding each other."* There is a documented **load order**, but it is not a winner-takes-all chain:

1. Broadest scope first, most specific last. Content is ordered from the filesystem root down to your working directory, so instructions closer to where you launched Claude are read **last**.
2. Within a directory, `CLAUDE.local.md` is appended **after** `CLAUDE.md`.

Reading last does not win a contradiction. The docs state plainly: *"if two rules contradict each other, Claude may pick one arbitrarily."* CLAUDE.md is delivered as a **user message**, not as part of the system prompt, and there is *"no guarantee of strict compliance."*

> **Do not confuse CLAUDE.md with `settings.json`.**
>
> | | Conflict behaviour |
> |---|---|
> | `CLAUDE.md` | Concatenated; contradictions may resolve **arbitrarily** |
> | `settings.json` | **Strict precedence:** managed policy > local > project > user. Managed policy always wins |
>
> If a question asks "which CLAUDE.md wins on a conflict?", the documentation-honest answer is **neither is guaranteed to** — move the rule to `settings.json` or a hook. Watch for distractors claiming "more specific scope wins" or "user-level overrides project-level." The official docs make neither claim.

Two consequences the exam tests repeatedly:

1. **Scope follows audience.** A team convention must live in a committed project file. Putting it in `~/.claude/CLAUDE.md` means it exists only on your laptop — a new teammate will never see it.
2. **Guidance is not enforcement.** Settings rules are enforced by the client regardless of what Claude decides; CLAUDE.md instructions only shape behaviour. If a rule must hold on every run — a blocked tool, a required formatter, a permission policy — it belongs in `settings.json` or a hook (see 1.5).

### `CLAUDE.local.md`

Lives next to `CLAUDE.md` at any level and loads the same way, with three differences:

- **Appended after** the `CLAUDE.md` at the same level (load order, not precedence).
- **Gitignored by convention** — the `.local` suffix flags files you don't commit.
- **For your quirks in this repo**, not the team's rules: a scratchpad path, a note you keep re-pasting, a temporary debugging reminder.

Think of it as a project-scoped `~/.claude/CLAUDE.md`. If you reach for it to express a *team* rule, that rule belongs in `CLAUDE.md` instead.

### Diagnosing what loaded: `/memory` and `/context`

| Command | What it does |
|---|---|
| `/memory` | Lists your `CLAUDE.md`, `CLAUDE.local.md`, and auto-memory locations, and opens any of them in your editor |
| `/context` | Reports what **actually loaded** into this session, under *Memory files* |

**Neither command loads anything.** Configuration loads automatically based on its level and location; these commands reveal what is already there. They diagnose, they do not activate — and that distinction is what the exam tests.

> **On the exam, answer `/memory`.** The exam guide (v1.0) predates the split and treats `/memory` as the command that shows which files are loaded. At a real keyboard, `/context` is what reports the loaded set.

**Trap:** "Developer B has not run `/memory` to load the configuration files" is a distractor in the official practice scenario. `/memory` never loads anything.

### What survives `/compact`

| Survives | Why |
|---|---|
| Project-root `CLAUDE.md` | Claude re-reads it from disk after compaction and re-injects it. It was never part of conversation history, so there was nothing for the summariser to compress |

| Does **not** come back automatically | Why |
|---|---|
| Nested subdirectory `CLAUDE.md` | Loads on demand — returns next time Claude reads a matching file |
| `.claude/rules/` with `paths:` frontmatter | Same |
| An instruction that only ever existed in conversation | Compaction is free to summarise it away |

When an instruction seems to vanish after `/compact`, this is usually why.

### Modular organisation with `@` path imports

> ⚠️ **There is no `@import` keyword.** The directive is just `@` followed by a path, on its own line. Much of the writing online — and an earlier version of these notes — gets this wrong.

```markdown
# .claude/CLAUDE.md

Coding standards:

@./standards/naming-conventions.md
@./standards/error-handling.md
@./standards/testing-requirements.md
```

Each `@<path>` line inlines that file into the CLAUDE.md at load time. Per-package CLAUDE.md files can pull in only the standards that apply to them — the API package imports API conventions, the frontend imports component rules, and nothing is duplicated.

**Imports load eagerly.** The referenced file is inlined the moment Claude reads your CLAUDE.md, exactly as if you had pasted it in. Splitting a 600-line CLAUDE.md into six 100-line imports makes the *source* nicer to maintain, but **the context Claude sees is the same size**. To shrink per-session context you need `.claude/rules/` with path-scoped frontmatter (3.3), which loads only for matching paths.

### The scenario

> A **15-package monorepo** has three standards documents: `/docs/standards/security-rules.md`, `testing-patterns.md`, and `api-conventions.md`. Their applicability varies by package domain. Today every package duplicates all three, so most packages carry irrelevant guidance. Package maintainers know their own domain requirements and want a decentralized, sustainable approach.

**Fix: each package's `CLAUDE.md` uses `@` lines for only the standard files relevant to its domain.**

```markdown
<!-- packages/payments-api/CLAUDE.md -->
# Payments API conventions

@../../docs/standards/security-rules.md
@../../docs/standards/api-conventions.md
```

Why this is the answer:

- **Single source of truth** — the standards live in one place and every importer picks up edits automatically.
- **Decentralized** — maintainers decide for their own package, matching the "maintainers know their domain requirements" hint.
- **Sustainable** — adding a package means adding its own small CLAUDE.md, not editing a central registry.

Why the alternatives lose:

| Distractor | Problem |
|---|---|
| `.claude/rules/` files with `paths:` frontmatter listing every package directory | Centralizes the decision in a path list that must be edited whenever packages are added, renamed, or moved — exactly the maintenance burden the question says to avoid |
| One shared file that every package imports | Reintroduces the duplication problem under a new name — every package gets all three standards again |
| All standards in the root CLAUDE.md with prose overrides ("ignore security-rules.md when...") | Brittle. Prose exceptions are guidance, not scoping, and they grow combinatorially |

**Imports vs. path rules** — know the distinction cold:

| | `@<path>` line | `.claude/rules/` with `paths:` |
|---|---|---|
| Loads | **Always**, inlined eagerly wherever referenced | **Only when matching files are touched** |
| Decided by | The file doing the importing | A glob pattern |
| Reduces context? | **No** — same tokens, nicer source | **Yes** |
| Best for | Composing a document from modular pieces, per-directory selection | A convention tied to a *file type* spread across many directories |

### The `.claude/rules/` directory

An alternative to one large CLAUDE.md: topic-specific rule files such as `testing.md`, `api-conventions.md`, `deployment.md`. Each may optionally carry YAML frontmatter with path scoping (3.3). **Without frontmatter, a rules file loads for all sessions** — the conditional loading comes from the `paths:` key, not from living in `.claude/rules/`.

### Don't post-process what you can specify upfront

> Claude generates a high volume of trivial tests that maximize line coverage rather than verify behaviour. Developers reject the suites. Post-generation filtering or a multi-pass scoring step would add latency, which the team cannot accept.

**Fix: document testing standards in CLAUDE.md** — concrete quality criteria, the available fixtures and their intended use cases, and examples contrasting a meaningful behavioural test with a trivial assertion.

The whole scenario hinges on the latency constraint. Each distractor is a *filtering* approach:

- *Restrict generation to directories where quality is already high* — treats a symptom and limits where you can work.
- *Post-generation coverage analysis that filters out low-quality tests* — catches bad tests after they're written; adds the latency you were told to avoid.
- *Two-phase generate-score-filter* — directly contradicts the constraint.

**Principle: when the constraint is "no added latency," the answer improves the first pass.** Give the model judgment criteria before it generates, not a filter after.

> **Moved out of 3.1:** the `REVIEW.md` scenario for suppressing automated code-review noise. It is a configuration file, but what it governs is a CI review job, so it now sits in [3.6](#36-cicd-integration).

### The signature 3.1 scenario

> Developer A has been on the team for months and Claude Code follows every team convention. Developer B joins, clones the repository, and gets inconsistent results that ignore those conventions. Same repo, same branch.

**Root cause: the conventions live in Developer A's `~/.claude/CLAUDE.md` instead of project-level config.** User-level configuration is not version-controlled and never travels through git.

**Fix: move them to `.claude/CLAUDE.md` or the root `CLAUDE.md`.**

Diagnose this on sight: **"new team member" + "inconsistent behaviour" → check where the configuration lives.**

### Exam traps for 3.1

- Putting team conventions in `~/.claude/CLAUDE.md`.
- Treating CLAUDE.md as a strict override chain — or claiming "more specific scope wins." Neither is guaranteed; `settings.json` is the layer with real precedence.
- Thinking `/memory` *triggers* loading. It is a diagnostic.
- Assuming a directory-level `CLAUDE.md` is right for a convention spanning many directories — that is path rules with globs.
- Choosing a filtering/post-processing answer when the scenario forbids added latency.
- Confusing `@` path lines (always load, eagerly, same token cost) with `.claude/rules/` + `paths:` (conditionally load).
- Writing `@import` as a keyword. It does not exist.

---

## 3.2 Custom slash commands and skills

### The unified skills system

Commands and skills have merged into one system. **Two locations both produce the same `/command`, but their file shapes differ** — and that difference is a direct exam trap:

| Path | Shape | Creates |
|---|---|---|
| `.claude/skills/deploy/SKILL.md` | A **directory** named after the command, with `SKILL.md` as the required entrypoint | `/deploy` |
| `.claude/commands/deploy.md` | A **flat Markdown file** whose filename becomes the command name | `/deploy` |
| `.claude/skills/deploy.md` | A flat file dropped into the skills directory | **Nothing.** Not picked up |

`.claude/skills/` is the **canonical** path. `.claude/commands/` still works for backward compatibility. Prefer skills: they add a supporting-files directory, automatic discovery so Claude can load one when it matches your intent, and precedence when a skill and a command share a name (**the skill wins**). Both paths support the same frontmatter.

### The core distinction

| | Always-on guidance | On-demand workflow |
|---|---|---|
| Mechanism | `CLAUDE.md`, matching path rules | Slash command, Skill |
| Cost | Tokens in **every** session | Descriptions are always in context; the **body** loads only when invoked |
| Good for | Conventions that apply broadly | Checklists, migrations, procedures used occasionally |

The recurring exam judgement: **does this content belong in every session, or only in sessions doing this specific job?**

**Invocation can be explicit or automatic.** You type `/skill-name`, or Claude loads a skill whose `description` matches your request, or whose `paths:` frontmatter matches the file you're editing. A skill with `disable-model-invocation: true` requires explicit invocation.

### Slash commands keep task-specific content out of unrelated sessions

> A team wants Claude to follow a detailed **8-item code review checklist** (API changes, test coverage, documentation, security, …) on pull requests. Claude is also used for feature work, debugging, and documentation generation. Developers currently paste the checklist at the start of each review session.

**Fix: create a `/review` slash command containing the checklist.**

- *Add it to CLAUDE.md under a "Code Review" heading* — the trap. It solves the pasting problem but now an 8-item review checklist occupies context during every debugging and feature session, where it is pure noise.
- *Make plan mode the default for reviews* — plan mode governs *whether Claude plans before acting*; it has nothing to do with loading a checklist.
- *Build a dedicated review subagent* — over-engineered. The need is "load this content when I ask for it," which is exactly what a command is.

### Project skills live in the repo

> A team has a step-by-step React-to-Vue component migration workflow. Every developer should invoke it as `/migrate-component`, and it must **stay in sync** as the team iterates on it.

**Fix: `.claude/skills/migrate-component/SKILL.md` at the project root, committed to version control.**

The two requirements map directly: *every developer* → committed to the shared repo; *stay in sync* → updates propagate on merge.

| Distractor | Why it fails |
|---|---|
| `~/.claude/skills/` on each developer's machine | Personal scope. Each copy drifts independently; no source of truth; a new hire gets nothing |
| A `skillOverrides` entry in `.claude/settings.json` | Not the registration mechanism — skills are discovered from the skills directory |
| An instruction block in the root CLAUDE.md | Content is shared, but there's no invocable `/migrate-component` command, and it loads in every session |

### Skill anatomy and frontmatter

```markdown
---
description: Analyse a feature area of the codebase and report structure, patterns and risks
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
argument-hint: "Specify the module path to analyse (e.g. src/api/auth)"
---

# Analysis steps
1. ...
```

The exam guide (v1.0) names **three** frontmatter keys. Two of them have drifted from their documented behaviour, and the guide's wording is still the keyed answer.

#### `context: fork`

Runs the skill in an **isolated sub-agent context**. Verbose output stays contained in the fork; the main conversation receives only the summary. Essential for:

- Codebase analysis (extensive file listings and code excerpts)
- Brainstorming (many alternatives and evaluations)
- Any task producing noisy, exploratory output

Without it, skill output flows into the main conversation, consumes context window tokens, and degrades the quality of subsequent responses. **When a scenario describes verbose skill output cluttering the conversation, the answer is `context: fork`.** This is the one key with no ambiguity between guide and implementation.

#### `allowed-tools`

| | Says |
|---|---|
| **Exam guide v1.0** | **Restricts** which tools the skill can access — **this is the keyed answer** |
| **Current Claude Code** | **Pre-approves** the listed tools so they run without a permission prompt. It does **not** restrict: every other tool remains callable, and normal permission settings still govern anything unlisted |

If a question asks what the field *does*, answer **"restricts."** But know the real boundary: to remove tools from Claude's pool while a skill runs, use **`disallowed-tools`**, or deny rules in permission settings. `allowed-tools` is not a security boundary.

#### `argument-hint`

| | Says |
|---|---|
| **Exam guide v1.0** | Prompts developers for required parameters when the skill is invoked without arguments — **this is the expected answer** |
| **Current Claude Code** | A hint shown during **autocomplete** to indicate expected arguments. It is a label, not an interactive prompt — invoking with no arguments does not stop and ask |

#### `description` — not on the guide's list, but the field that does the most work

It is what Claude reads to decide whether a skill applies to your request. **A skill without a useful description only ever runs when you type `/name` yourself.** Omit it and Claude Code falls back to the first paragraph of the skill body, which is usually a worse summary than one you would write.

#### Other live frontmatter keys

`disallowed-tools` · `disable-model-invocation` · `model` · `effort` · `when_to_use` · `paths`

### Where things live

| Need | Location | Scope |
|---|---|---|
| Team-wide command | `.claude/skills/<name>/SKILL.md` (or `.claude/commands/<name>.md`) | Project — shared via git |
| Personal command | `~/.claude/skills/<name>/SKILL.md` (or `~/.claude/commands/<name>.md`) | User — not shared |
| Universal standards | `.claude/CLAUDE.md` or root `CLAUDE.md` | Project — always loaded |
| Personal preferences | `~/.claude/CLAUDE.md` | User — not shared |

**Personal variants:** if the team has a standard `/analyse` skill but you want a more verbose one, create it in `~/.claude/skills/` under a *different* name (`/deep-analyse`). It won't override or conflict with the team version.

> **The scoping pattern is consistent across all of Domain 3:** `.claude/` is project-scoped and shared via git; `~/.claude/` is personal. This holds for CLAUDE.md, commands, skills, and rules alike.

### Exam traps for 3.2

- Dropping a flat `.md` file into `.claude/skills/` and expecting a command. Skills are **directories with a `SKILL.md` inside**; flat files belong in `.claude/commands/`.
- Placing a team-shared command in `~/.claude/`.
- Putting task-specific checklists or procedures in CLAUDE.md where they load for every unrelated task.
- Expecting a skill to behave like CLAUDE.md for always-on guidance. If the question asks about conventions applying to **every** edit, the answer is CLAUDE.md or `.claude/rules/`.
- Not recognizing `context: fork` when a scenario describes verbose output polluting the conversation.
- Treating `allowed-tools` as the security boundary — that is `disallowed-tools` or a deny rule.
- Expecting a CLAUDE.md block to produce an invocable command.

---

## 3.3 Path-specific rules for conditional convention loading

### The mechanism

Files in `.claude/rules/` carry YAML frontmatter with a `paths` glob. The rule loads **only when a matching file is being worked on**.

```markdown
---
paths: ["terraform/**/*"]
---
# Terraform conventions
- Every module must expose `outputs.tf` with documented descriptions.
- Pin provider versions explicitly; never use `~>` for major versions.
```

### The scenario

> An infrastructure-as-code repository contains Terraform modules (`/terraform/`), Kubernetes manifests (`/kubernetes/`), and CI/CD pipeline scripts (`/pipelines/`), each with different conventions. The single root `CLAUDE.md` has grown past **500 lines**. When a developer edits Kubernetes files, all the Terraform-specific rules load too, consuming context tokens for no benefit.

**Fix: split the conventions into `.claude/rules/` files with `paths:` frontmatter** — `terraform.md` scoped to `terraform/**/*`, `kubernetes.md` to `kubernetes/**/*`, and so on. Only the relevant rules enter context.

Why each alternative misses:

| Distractor | What it fixes | What it doesn't |
|---|---|---|
| Restructure the root CLAUDE.md with clear headers | Human readability | Every section still loads; token cost unchanged |
| Use `@path/to/import` syntax from the root CLAUDE.md | Modular authoring | Imports load **unconditionally** — the root file still pulls everything in |
| Split into per-subdirectory `CLAUDE.md` files | Partially works for directory-aligned conventions | Ties rules to folder location; can't express cross-cutting patterns like `**/*.test.ts` that span many directories |

That last point is the sharpest distinction in the subtopic. **Directory CLAUDE.md files scope by location; path rules scope by glob.** When a convention follows a *file type* rather than a *folder*, only globs work.

### The three scoping mechanisms side by side

| Mechanism | Loads when | Scoped by |
|---|---|---|
| Root `CLAUDE.md` | Always | Nothing |
| Subdirectory `CLAUDE.md` | Working in that subtree | Directory location |
| `@<path>` line | Whenever the importing file loads | Not conditional at all |
| `.claude/rules/` **without** frontmatter | Always | Nothing |
| `.claude/rules/` **with** `paths:` | Matching files are touched | **Glob pattern** |

**If the question mentions token efficiency, context bloat, or a bloated root CLAUDE.md, the answer is path rules.**

---

## 3.4 Plan mode vs. direct execution

### The decision rule

**Decide on ambiguity and blast radius — not on difficulty.**

A hard problem with one obvious location goes straight to execution. An easy-sounding change touching 45 files does not.

| Go direct when | Plan first when |
|---|---|
| One file, one function | Many files, cross-module |
| A stack trace names the location | You don't know what's affected |
| Mechanical and well-scoped | Multiple viable architectures |
| Easily reverted | Breaking changes |

### Direct: a small, contained change

> Add a date validation check ensuring event dates are in the future — a conditional in one existing function in a single file.

**Direct execution.** Nothing is ambiguous and nothing else is affected. Plan mode here is process overhead with no corresponding reduction in risk. (Extended thinking is likewise unnecessary for mechanical logic.)

### Direct: a stack trace is already a plan

> A critical production bug has a clear stack trace pointing at a specific function in an **unfamiliar** `OrderProcessing` module. The error logs are specific and actionable.

**Direct execution — follow the stack trace to the code and fix it.**

The word "unfamiliar" is bait. It makes "enter plan mode to explore the module architecture and dependencies" sound prudent. But **the stack trace has already done the localization work** that plan mode would do, and it's a critical production bug where the overhead costs real time. Unfamiliarity with a module is handled by reading the code the trace points to, not by surveying the architecture.

### Plan: multiple viable architectures

> Adding real-time updates to an application. The options are WebSockets, Server-Sent Events, or polling — each with different complexity, browser support, and infrastructure requirements.

**Plan mode** — explore the architecture, evaluate the trade-offs, and present options for team approval before writing code.

There is no single correct answer determinable from the code; the decision depends on constraints that need to be surfaced and agreed. Starting direct execution on one approach commits you before the comparison happens, and the cost of reversing an architectural choice is high.

### Plan: large-scale breaking changes

> A security audit requires upgrading an authentication library from v2 to v3. The migration guide lists breaking changes: `authenticate()` now returns a Promise instead of taking a callback, the `User` type has restructured fields, and three deprecated methods were removed. Grep shows **45 files across several modules** import the library.

**Plan mode** — explore usage across all 45 files, map the affected code paths against each breaking change, then produce a migration strategy before implementing.

- *Paste the migration guide and execute directly across all 45 files* — assumes every call site has the same shape. Callback-to-Promise conversions in particular vary wildly by context (nested callbacks, error-first handlers, mixed sync paths). A partial migration leaves the codebase broken.
- *Update the dependency, run the tests, fix failures iteratively* — only finds what the tests cover, and on a security-driven migration an untested path that silently misbehaves is the worst outcome.
- *Write a slash command for the migration without exploring first* — same uniformity assumption, now automated.

> **Moved out of 3.4:** the "have Claude interview you" scenario. It was filed here because it looks like a planning decision, but the gap it addresses is unknown **requirements** rather than unknown code. Eliciting requirements is a refinement technique — see [3.5](#35-iterative-refinement-techniques). The distinction is still a 3.4 decision, so it is summarized below.

### The boundary of plan mode

| The unknown is… | Remedy | Why not the other |
|---|---|---|
| Which code is affected | **Plan mode** | — |
| Whether evidence already localizes it | **Direct execution** | Planning repeats work the evidence did |
| What correct behaviour even is | **Have Claude interview you** (3.5) | Plan mode explores what *exists*; it cannot surface a requirement nobody recorded |

### Exam traps for 3.4

- Choosing plan mode because a problem sounds hard or a module is unfamiliar, when the location is already known.
- Choosing direct execution on a multi-file breaking migration because a migration guide exists.
- Choosing plan mode when the real gap is domain knowledge rather than code knowledge.

---

## 3.5 Iterative refinement techniques

Twelve questions — the densest subtopic in D3, though half are duplicate pairs. One theme runs through all of them: **concrete, objective, and checkable beats descriptive.**

### Close the knowledge gap before writing code

> You're implementing a caching layer for API responses on a `/products` endpoint using Redis with a 5-minute TTL — but you are **unfamiliar with production caching requirements**. You're uncertain about invalidation strategies, cache layering, consistency guarantees, and failure modes.

**Fix: ask Claude to interview you to surface production caching considerations before implementation begins.**

The gap isn't "which files are affected" — it's **you don't know what you don't know**. Plan mode analyzing the existing endpoint tells you about the code you already have; it cannot surface a requirement that was never written down.

- *Use plan mode to analyze the current endpoint implementation* — analyzes existing code; doesn't surface unknown requirements.
- *Start minimal and iterate through testing failures* — an invalidation bug may never surface in testing, and will surface in production.
- *Write a specification with TBD markers* — requires you to already know which areas are unknown. The whole problem is that you don't.

**Rule: unknown files → plan mode. Unknown *requirements* → interview first.** This belongs with the refinement techniques because it is the same move as every other one in this subtopic: replace an abstraction with something concrete before acting.

### Prefer an objective signal: write tests first

> Implementing a complex graph traversal algorithm with specific performance requirements and edge cases — disconnected nodes, cycles, weighted edges.

**Fix: write a test suite covering the behaviour, edge cases, and performance requirements *before* implementation. Ask Claude to write code that passes, then iterate by sharing the test failures.**

The tests give every iteration an objective, automatically-checkable signal. You never have to judge "is this handling cycles correctly?" — the suite answers it.

- *A detailed natural-language spec plus manual review each round* — subjective feedback, and edge cases slip through when you're eyeballing diffs.
- *Have Claude rewrite a reference implementation and match the style* — no verification at all.
- *Extensive research and a detailed plan before implementation* — reasoning without a feedback signal. Edge cases still escape.

### Prefer a concrete failing case: don't describe the bug

> Claude's data migration script doesn't correctly handle records with null values in required fields.

**Fix: hand Claude a concrete test case — the null-value input and the expected output — and ask it to fix that.**

A failing case is unambiguous. Prose descriptions ("it should handle nulls gracefully") leave the target to interpretation.

- *Describe the null problem in detail and ask for a full regeneration* — without a concrete anchor, the same edge case or a neighbouring one slips through again, and you've thrown away working code.
- *Manually edit the generated code yourself* — breaks the feedback loop; the next generation reintroduces the bug.
- *Add "think harder" to the prompt* — vague, anchors nothing.

### Prefer concrete code: use `@` references

> Implementing a new payment processing module that must follow established patterns for database transactions, error handling, and audit logging. Three existing modules exemplify these patterns: `db_utils.py`, `error_handlers.py`, `audit_logger.py`. This is a **one-off** integration task.

**Fix: `@`-reference the three modules directly in the prompt.**

Real code carries the subtle conventions — naming, error wrapping, parameter ordering, logging structure — that paraphrase inevitably loses.

- *Describe the patterns in natural language* — lossy. You will omit something you don't consciously notice.
- *Ask Claude to explore the codebase and find the patterns* — wasteful when you already know exactly which three files matter.
- *Add the patterns to CLAUDE.md* — the **"one-off" tell**. CLAUDE.md is for recurring project-wide conventions; a single integration task doesn't warrant permanent context cost.

**Note the consistent ranking: concrete example > prose description, every time.** And note the scoping judgement: one-off → put it in the prompt; recurring → put it in CLAUDE.md.

### Fix interacting issues one at a time

> A PDF report feature has three formatting problems: column widths are too narrow (content truncates), dates render unformatted, and page breaks land incorrectly. **They interact** — changing column widths affects date rendering, and page breaks depend on content height.

**Fix: fix and test one issue at a time.**

With interacting changes, a bundled fix makes attribution impossible: if the output is still wrong, you cannot tell which of three changes caused it, and one change can mask another's regression.

- *Bundle all three in one message* — fast if it works, undebuggable if it doesn't.
- *Show a correct example report and ask Claude to match it* — doesn't tell you which fix to apply or in what order, and the interactions remain unanalyzed.
- *Start fresh with all requirements upfront* — the interactions are still there; you've just moved them.

**Contrast with 4.x guidance to batch related feedback.** The rule is: **changes that affect each other → sequence them one at a time; independent changes → batch them.** Here they explicitly interact, so sequence.

### Use a multi-phase workflow only where ambiguity justifies it

> Two requests: **(A)** "Rename `getUserData` to `fetchUserProfile` everywhere." **(B)** "Improve error handling throughout the data processing module — add try/catch, produce meaningful errors, ensure failures don't silently corrupt data." Which benefits more from an analyze → propose → implement → review workflow?

**Request B.**

- **A** is a mechanical find-and-replace with a verifiable end state. Phases add overhead and nothing else.
- **B** is broad and genuinely ambiguous: *which* failures matter, what counts as a "meaningful" error, where silent corruption is possible. Analysis surfaces the failure modes, a proposal lets you agree on the approach before code is written, and review catches what was missed across modules.

**Rule: structured multi-phase workflows pay for themselves in proportion to ambiguity.** Apply them to open-ended quality work; skip them for mechanical transformations.

### The refinement toolkit, ranked

| Situation | Best tool |
|---|---|
| Complex behaviour with edge cases | Test suite written first; iterate on failures |
| A specific known bug | Concrete failing test case |
| Must match existing conventions (one-off) | `@` references to the real files |
| Must match existing conventions (recurring) | Document in CLAUDE.md |
| Several interacting defects | One at a time, test after each |
| Broad, ambiguous quality work | analyze → propose → implement → review |
| Mechanical rename/transformation | Just do it |

### Exam traps for 3.5

- Choosing a prose description when a concrete example or failing test is offered.
- Bundling fixes that the scenario explicitly says interact.
- Adding one-off task context to CLAUDE.md.
- Applying heavyweight multi-phase process to a mechanical change.

---

## 3.6 CI/CD integration

### `-p` is the single most tested fact in Domain 3

`claude -p "<prompt>"` (equivalently `--print`) switches Claude Code to print mode: it processes the prompt, writes the result to stdout, and exits. Interactive mode expects keyboard input; a CI runner has no keyboard, so **without `-p` the job hangs forever.**

```bash
claude "Analyse this pull request"      # WRONG — hangs in CI
claude -p "Analyse this pull request"   # CORRECT
```

This is pure memorisation, and it is **Question 10 in the official sample question set**. The exam shows a hanging CI job with logs of Claude awaiting input and asks for the fix. The distractors are all inventions:

| Distractor | Status |
|---|---|
| `CLAUDE_HEADLESS=true` | **Does not exist** |
| `--batch` | **Does not exist** |
| Redirect stdin from `/dev/null` | Does not properly address interactive mode |

### The full CLI flag surface

**System prompt — four flags, and the append/replace distinction is tested:**

| Flag | Effect |
|---|---|
| `--system-prompt "<text>"` | **Replaces** the entire default system prompt |
| `--system-prompt-file <path>` | **Replaces** it with a file's contents |
| `--append-system-prompt "<text>"` | **Appends** text to the default prompt |
| `--append-system-prompt-file <path>` | **Appends** a file's contents |

**Append** when Claude should stay a coding assistant that also follows your extra rules — you keep the default tool guidance, safety instructions, and coding conventions, and supply only what differs. **Replace** only when the identity or permission model genuinely differs from Claude Code's, such as a non-coding agent in an unwatched pipeline. Replacing drops everything, so you now own every behaviour the task still needs.

**Headless output and limits (print mode):**

| Flag | Effect |
|---|---|
| `--output-format text\|json\|stream-json` | Output shape for `-p`; `json` and `stream-json` are machine-parseable |
| `--input-format text\|stream-json` | Input shape for `-p` |
| `--json-schema '<schema>'` | Schema-validated output for `-p` |
| `--max-turns <n>` | Cap the number of agentic turns, then exit |

> **Where schema output lands:** with `--output-format json`, the validated data appears in the envelope's **`structured_output`** field. Extract it with `jq '.structured_output'` — **not** from the top level. The envelope also carries result text, session ID, and cost/usage metadata.

**Permissions, tools, and context:**

| Flag | Effect |
|---|---|
| `--permission-mode <mode>` | `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`, or `manual` (alias for `default`, v2.1.200+) |
| `--allowedTools "<rules>"` | Tools that run **without a permission prompt**, e.g. `"Bash(git diff *)" "Read"` |
| `--disallowedTools "<rules>"` | Deny rules. A **bare tool name removes the tool from context entirely** |
| `--tools "Bash,Edit,Read"` | Restrict which built-in tools are available at all |
| `--add-dir <path>` | Add a directory Claude may read and edit (grants **file access**, not configuration discovery) |
| `--model <alias\|name>` | Set the session model (`sonnet`, `opus`, or a full model name) |

**Session and start-up:**

| Flag | Effect |
|---|---|
| `-c` / `--continue` | Resume the most recent conversation in the current directory |
| `-r` / `--resume <id\|name>` | Resume a specific session |
| `--bare` | Minimal mode: skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md. Leaves Claude with **Bash and file read/edit tools only** |

> ⚠️ **A note on `--max-budget-usd`.** One bank question keys on `--max-turns` *and* `--max-budget-usd` together as caps "enforced by Claude Code itself." The official lesson's flag reference lists **`--max-turns` only**. Treat `--max-turns` as the reliable answer for an iteration cap; if a spend cap appears as an option alongside a job-runner timeout, the Claude-Code-side flag is still the better choice, but do not expect `--max-budget-usd` to be the keyed term on a flag-recall question.

### Append vs. replace: the most-tested flag distinction

> A CI code review pipeline analyzes Git diffs via stdin with custom review instructions. Claude reads only the raw diff text and never inspects related files — for example, a modified core function called by many external modules gets reviewed in isolation. The pipeline needs file-reading and code navigation.

**Cause: `--system-prompt` replaced Claude's entire default system prompt, stripping the built-in guidance about *when and why* to use file-reading and navigation tools.**

**Fix: use `--append-system-prompt` (or `--append-system-prompt-file ./CLAUDE.md`)** so the custom review instructions sit on top of the default prompt rather than replacing it.

The critical distractor: *keep `--system-prompt` but add `--allowedTools "Read, Glob, Grep"`.* This is the one to understand properly.

> **Permission to use a tool is not the same as knowing when to use it.**

`--allowedTools` makes the tools *available*. The default system prompt is what tells Claude that when reviewing a change to a shared function it should go read the call sites. Replace that prompt and the tools sit there unused.

The other distractors — piping the diff differently, or moving instructions into CLAUDE.md — leave the replaced system prompt in place, so the root cause survives.

**Default to `--append-system-prompt`. Reach for `--system-prompt` only when you deliberately want to discard built-in behaviour.**

### Hard caps belong inside Claude Code

> Large pull requests (>50 changed files) trigger extensive agentic loops running over 20 minutes and costing **$8–$12 per run**. You must enforce both an iteration cap and a budget cap, **enforced by Claude Code itself** rather than by an external runner.

**Fix: `claude -p ... --max-turns 10 --max-budget-usd 2.00`.** Both flags are enforced inside the invocation and abort the loop the moment either cap is hit.

| Distractor | Why it fails |
|---|---|
| Switch to a smaller, cheaper model | Lowers per-token cost; enforces no cap at all. A runaway loop on a cheap model is still a runaway loop |
| GitHub Actions `timeout-minutes` plus Console monitoring | Enforcement at the **job-runner** level — it kills the process, leaving no result and no graceful termination. Monitoring is after the fact. The question explicitly asks for enforcement by Claude Code |
| `--permission-mode dontAsk` | Controls tool permission prompting. Nothing to do with turns or spend |

**Read the qualifier.** "Enforced by Claude Code itself" exists specifically to eliminate the job-timeout answer, which otherwise looks reasonable.

### `--bare` for CI startup latency

> An automated code review CI job spends **18 seconds** initializing before Claude begins analyzing code. Profiling shows the delay comes from automatically discovering hooks, MCP servers, plugins, skills, and nested `CLAUDE.md` files throughout a large monorepo. The job must still enforce the coding standards from the root `CLAUDE.md`.

**Fix: run with `--bare` and pass `--append-system-prompt-file ./CLAUDE.md`.**

Two requirements, two parts of the answer: `--bare` skips the auto-discovery causing the 18 seconds; `--append-system-prompt-file` loads the one file you actually need, explicitly, while preserving the default prompt.

| Distractor | Why it fails |
|---|---|
| `--bare` with `--system-prompt-file ./CLAUDE.md` | Loads the rules but **replaces** the default prompt assembly, losing built-in tool guidance — the same mistake as the previous question |
| `--bare` with the review criteria repeated in the `-p` prompt | Fixes startup but duplicates a version-controlled file into a CI script, where it will drift |
| Keep default initialization and add `--exclude-dynamic-system-prompt-sections` | Trims prompt sections; does not skip the auto-discovery that is actually causing the delay |

### Review in a separate invocation

A recurring principle: **the session that wrote the code is biased toward its own assumptions.** When Claude generates code it accumulates reasoning context — why it chose this approach, what it considered, what it rejected. Ask it to review that same code in the same session and it keeps all of it, making it less likely to question decisions it already justified to itself.

```bash
claude -p "Implement the authentication middleware"     # session A
claude -p "Review the authentication middleware for \
  security issues, error handling gaps, and edge cases"  # session B — independent
```

This is a measurable effect, not a theoretical worry. (Same idea as 4.6's multi-instance review.)

### Incremental review context

Automated reviews run on every push. Without context from previous runs, each one analyses the whole PR from scratch and **re-derives the same findings every time**.

The subtlety: a genuinely fixed issue drops out on its own, because the changed code no longer triggers it. The findings that keep reappearing are the ones the developer saw and **deliberately chose not to change** — and a context-free re-scan cannot tell those apart from new problems.

**Fix: include prior findings in context and instruct Claude to report only (1) new issues and (2) issues from the previous run that are still present** — explicitly not re-reporting what the developer already reviewed and declined to act on.

Why it matters: duplicate comments erode developer trust. When the same five comments appear on every push regardless of fixes, developers stop reading them.

### CLAUDE.md is read in CI too

Claude Code reads the project's CLAUDE.md files in CI exactly as it does interactively, so CLAUDE.md is how you feed project context to a CI-invoked run: testing standards, available fixtures and what data they contain, review severity criteria, and existing coverage. Without it, CI-invoked test generation produces low-value boilerplate.

Relatedly: **include existing test files in context** when generating tests in CI, or Claude will suggest tests that already exist.

### Batch vs. real-time for CI workflows

| Workflow | Choice | Why |
|---|---|---|
| Pre-merge checks | **Real-time** | Blocking — developers wait on the result |
| Overnight technical-debt reports | **Batch** | Not time-sensitive; 50% saving |
| Weekly code audits | **Batch** | Scheduled, latency-tolerant |
| Nightly test generation | **Batch** | Runs overnight, reviewed next morning |

The Message Batches API offers ~50% savings with processing up to 24 hours and **no latency SLA**, which disqualifies it for anything blocking. The exam tests this directly (Sample Question 11).

### Beyond the task statement: GitHub Actions

> Task Statement 3.6 never names GitHub Actions, but multiple exam reports describe four or five questions on it.

Anthropic ships the integration as **`anthropics/claude-code-action@v1`**, which runs Claude Code non-interactively inside a workflow job. It is built on the Agent SDK and accepts the same CLI flags.

**The `prompt` input decides the mode:**

| `prompt` | Behaviour |
|---|---|
| Absent | **Interactive mode** — waits for a trigger phrase (`@claude` by default) in an issue or PR comment and answers there |
| Present | **Automation mode** — runs on whatever event fired the workflow, without waiting for a mention, writing to the run log unless told to post elsewhere |

**CLI flags go through `claude_args`** as one string — there is no separate action input for the system prompt, tool allowlist, or turn cap:

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    claude_args: |
      --append-system-prompt "Review against the checklist in CLAUDE.md."
      --allowedTools "Read,Grep,Glob,Bash(git diff *)"
      --max-turns 10
```

Points worth knowing:

- With a plain-text prompt, Claude has **no shell or GitHub API access** until the workflow grants tools via `--allowedTools` in `claude_args` or a `permissions.allow` rule in the `settings` input.
- `actions/checkout` is what puts the repository on the runner. Without it there is nothing on disk to read.
- The `permissions` block bounds what Claude can write back. **`id-token: write`** is required for default GitHub App authentication; **`actions: read`** lets it see CI results on the PR.
- Credentials are repository secrets — `anthropic_api_key`, or `claude_code_oauth_token` for a subscription token. Never a literal in the workflow file.
- Pre-v1 input mapping: `custom_instructions` → `--append-system-prompt`, `direct_prompt` → `prompt`, and `mode` is gone because the action now infers it.

### Beyond the task statement: parallel sessions with git worktrees

A git worktree is a second working directory for the same repository, on its own branch, sharing history. Running each Claude Code session in its own worktree means edits in one never touch files in another — which is how several instances can refactor the same repo simultaneously.

| Command | Effect |
|---|---|
| `claude --worktree <name>` | Creates one under `.claude/worktrees/<name>/` on a branch `worktree-<name>`, and starts the session inside it |
| `git worktree add ../project-feature -b feature` | The manual equivalent, with your own path and branch |

Each session gets **its own full context budget** as well as its own files. Coordination on a shared file is **sequencing, not locking**: the first session merges, the second rebases onto the result before touching it. A custom subagent can run in its own worktree via `isolation: worktree` in its frontmatter.

### Shaping what an automated review reports

> Claude's managed GitHub App Code Review produces **18 findings per PR**, of which only about **4** are genuine logic bugs. The noise falls into three buckets: (1) style/formatting already enforced by CI linters, (2) findings on auto-generated template code under `src/gen/*`, (3) rendering-helper patterns that are intentional but superficially resemble anti-patterns.

**Fix: create `REVIEW.md` at the repository root** with explicit skip rules for CI-enforced checks and for generated paths, plus a requirement that rendering-related findings **cite a specific line demonstrating incorrect behaviour**.

Each of the three noise sources gets a targeted treatment:

| Noise source | Rule |
|---|---|
| CI-enforced style | Skip — the linter already owns this |
| `src/gen/*` generated code | Skip — the code is not human-authored |
| Intentional rendering patterns | Require evidence: cite a line showing actual incorrect behaviour |

That third rule is the clever part — you don't ban the category (some rendering findings *are* real), you raise the evidentiary bar so speculative ones get filtered out. **When you can't cleanly exclude a category, make it prove itself.**

Why `REVIEW.md` beats the alternatives: it's **version-controlled, discoverable, and durable** — any engineer can find it, review it, and change it through a PR. A `prompt` parameter buried in a GitHub Actions workflow expresses similar rules but is far less discoverable. Separate workflows per code area add CI infrastructure without addressing the three specific noise sources. Explanatory prose in CLAUDE.md provides context but no actionable skip rules and no evidence requirement.

> This question was previously filed under 3.1 because `REVIEW.md` is a configuration file. What it configures is the CI review job, which is why it belongs here.

### Exam traps for 3.6

- **A CI job hanging on interactive input.** The fix is `-p` / `--print`. Not `CLAUDE_HEADLESS=true`, not `--batch`, not stdin redirection — the first two do not exist.
- Using `--system-prompt` in CI and losing built-in tool guidance.
- Treating `--allowedTools` as a substitute for the guidance that says *when* to use tools.
- Reading `--json-schema` output from the top level instead of the envelope's `structured_output` field.
- Choosing a job-runner timeout when the question asks for enforcement inside Claude Code.
- Choosing a cheaper model as a "cost cap."
- Using the Batch API for a **pre-merge** check — it is blocking, and batch has no latency SLA.
- Assuming same-session self-review is as effective as an independent review instance.
- Omitting prior findings from a repeat review run, producing duplicate comments that erode developer trust.
- Reaching for extra CI infrastructure when a discoverable, version-controlled config file addresses the specific causes.

> **Moved out of 3.6:** the invoice `calculated_total` vs. `stated_total` consistency question. It was set in a pipeline, but the concept is extracting a derived value alongside a stated one and flagging the mismatch — see [4.4](../Domain-4-Prompt-Engineering-Structured-Output/summary.md#44-validation-retry-and-feedback-loops).

---

## The eight sentences that answer most of Domain 3

1. **Scope follows audience.** `.claude/` is shared via git; `~/.claude/` is personal. This holds for CLAUDE.md, commands, skills, and rules alike.
2. **CLAUDE.md is concatenated guidance, not precedence.** Contradictions may resolve arbitrarily. `settings.json` is the layer with a real chain (managed > local > project > user) and real enforcement.
3. **Load conditionally when you can.** Path rules for file-type conventions; skills and slash commands for occasional workflows; CLAUDE.md only for what's always relevant. `@` lines do **not** reduce context — they inline eagerly.
4. **Plan on ambiguity and blast radius, not difficulty.** And when the unknown is *requirements* rather than *code*, have Claude interview you.
5. **Concrete beats descriptive.** A failing test or an `@`-referenced file outperforms any amount of prose.
6. **Append, don't replace.** `--append-system-prompt` preserves the built-in tool guidance that makes Claude Code work.
7. **`-p` for anything non-interactive.** The most directly tested fact in the domain.
8. **Diagnostics don't activate.** `/memory` and `/context` reveal what already loaded; configuration loads on its own.

## Study order within this domain

Because the bank under-covers D3 and duplicates heavily within it, work through the official lessons in this order:

1. **3.1 CLAUDE.md hierarchy & scoping** — 4 questions; the three levels, `CLAUDE.local.md`, load order vs. precedence, `@` path lines, `/memory` vs. `/context`, and what survives `/compact`.
2. **3.3 path-specific rules** — 2 questions, both the same scenario, but a full blueprint subtopic; learn the frontmatter and the glob-vs-directory distinction.
3. **3.2 slash commands & skills** — 3 questions; the directory-vs-flat-file shapes, and the three frontmatter keys with their guide-vs-current splits.
4. **3.6 CI/CD** — 3 questions against a large flag surface; memorize the flag tables, it's pure recall and the cheapest points in the domain.
5. **3.4 plan mode** — 6 questions; practice the ambiguity/blast-radius call on borderline cases.
6. **3.5 iterative refinement** — 12 questions, best-covered; reinforce the "concrete beats descriptive" ranking.
7. **[3.0 cross-topic questions](../../categorized_exam_dump/domain-3-claude-code-configuration-workflows/3.0-cross-topic-and-domain-level.md)** — eight authored scenarios, weighted toward the thin subtopics this bank under-serves.

### Where the guide and current Claude Code disagree

For these, **answer the guide** and know the reality:

| Item | Guide (keyed answer) | Current behaviour |
|---|---|---|
| `allowed-tools` | **Restricts** the skill's tool access | **Pre-approves** listed tools; `disallowed-tools` is the real boundary |
| `argument-hint` | Prompts for required parameters | An autocomplete label; does not stop and ask |
| Diagnostic command | **`/memory`** shows what loaded | `/context` reports the loaded set; `/memory` lists and opens files |

## Links

[3.1 CLAUDE.md hierarchy](https://claudecertificationguide.com/learn/3-claude-code-config/3-1-claude-md-hierarchy) ·
[3.2 Slash commands & skills](https://claudecertificationguide.com/learn/3-claude-code-config/3-2-slash-commands-skills) ·
[3.3 Path-specific rules](https://claudecertificationguide.com/learn/3-claude-code-config/3-3-path-specific-rules) ·
[3.4 Plan mode vs. execution](https://claudecertificationguide.com/learn/3-claude-code-config/3-4-plan-mode-execution) ·
[3.5 Iterative refinement](https://claudecertificationguide.com/learn/3-claude-code-config/3-5-iterative-refinement) ·
[3.6 CI/CD integration](https://claudecertificationguide.com/learn/3-claude-code-config/3-6-cicd-integration)
