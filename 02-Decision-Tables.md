# Decision Tables — by Domain and Subtopic

Every "given X, choose Y" table from the domain notes, collected in one place. Use this for drilling and for last-pass revision; use the domain files for the reasoning and the worked examples.

**Reading the tables:** the left column is a *condition you can detect in a scenario*, not a phrase to match. Mechanism names, flags, and API fields are given exactly because those are product facts, not question wording.

**A caution about subtopic headings.** The tables are grouped by subtopic for lookup convenience, but the exam does not label its scenarios. Several concepts legitimately sit in two places — enforcing a monetary threshold is both a workflow rule and a tool-interface decision; a tool schema consuming the token budget is both tool design and context management. Where that happens, the concept appears under both headings on purpose. **Match on the structure of the problem, never on which section you found it in.**

**Jump to:** [D1](#domain-1--agentic-architecture--orchestration) · [D2](#domain-2--tool-design--mcp-integration) · [D3](#domain-3--claude-code-configuration--workflows) · [D4](#domain-4--prompt-engineering--structured-output) · [D5](#domain-5--context-management--reliability) · [Cross-domain](#cross-domain-tables)

---

# Domain 1 — Agentic Architecture & Orchestration

## 1.1 Agentic loops

### Loop control

| Question being asked | Answer |
|---|---|
| What decides which tool is called next? | The tool result enters context as a `tool_result`; **the model reasons** over it |
| What decides whether the loop continues? | The **`stop_reason`** field on each API response |
| Continue the loop when… | `stop_reason == "tool_use"` |
| Exit the loop when… | `stop_reason` is terminal: `"end_turn"`, `"max_tokens"`, `"stop_sequence"` |
| What is `max_turns` for? | A **safety ceiling** on runaway loops — never a completion signal |

### What the loop is *not*

| Proposed mechanism | Why it isn't the answer |
|---|---|
| A tool sequence planned at the start | Loops are reactive; step 3 depends on step 2's result |
| A decision tree mapping attributes to tools | That's a rules engine, and it can't handle unenumerated cases |
| Orchestration-layer routing on a result field | The orchestration layer executes and shuttles; it doesn't infer policy |
| Presence of a text block as the stop signal | Text and `tool_use` blocks coexist in one message |
| A tool-call counter as the stop signal | Fires on runaway, says nothing about completion |

---

## 1.2 Multi-agent orchestration

### Delegate or not

| Condition | Choice |
|---|---|
| The coordinator already holds the needed context | **Answer directly** — don't spawn |
| The work needs specialization or context isolation | Delegate via `Task` |
| Request complexity varies and the mix keeps shifting | Let the **coordinator decide per request** which subagents to invoke |

### Execution shape

| Condition | Choice |
|---|---|
| Subtasks are independent | Emit **all `Task` calls in one response message** → concurrent |
| Each subtask informs the next | Sequential turns |
| You want parallelism | Change the **message structure**, not the prompt wording |

### Diagnosing a delegation that didn't happen

| Observable | Cause |
|---|---|
| Narrates delegating, no invocation, **no errors** | `"Task"` missing from allowed tools |
| Partial / malformed `tool_use` blocks in logs | `max_tokens` truncation |
| Subagent runs but has nothing to work with | Context not forwarded → see 1.3 |
| Never even reasons about delegating | Subagent types never described in the system prompt |

### Closing a one-way pipeline

| Required part | Why all three are needed |
|---|---|
| Downstream agent reports a **specific named gap** | A general "something's missing" signal can't be acted on |
| Coordinator **re-delegates targeted** work | Planning up front can't anticipate post-analysis discoveries |
| Coordinator **re-verifies** the gap closed | Without this the loop has no termination criterion |

---

## 1.3 Subagent invocation and context passing

### Hand-off pattern by payload size

| Condition | Pattern |
|---|---|
| Small outputs, few agents | Inline the findings in the downstream prompt |
| Large outputs (tens of thousands of tokens) | Reference IDs + a shared store the subagent is **explicitly configured** to read |
| Any size | ❌ Never message queues with subscriptions, never direct agent-to-agent calls |

### Diagnosing a subagent that produced nothing useful

| Symptom | Cause | Fix |
|---|---|---|
| "No findings were provided" — **total absence** | Missing explicit hand-off | Put the data in the prompt |
| Partial findings, early content missing | Capacity | Reduce or reference rather than inline |
| Generic language replacing specifics | Attention dilution | Summarize + spawn a focused subagent |

| Proposed cause | Why it's wrong |
|---|---|
| "Context window too small" | Produces truncation, never total absence |
| "They need a shared API connection" | Transport ≠ context |
| "Give it a tool to fetch the other agent's history" | No such addressable store exists unless you built one |

### How to instruct a subagent

| Condition | Instruction style |
|---|---|
| Subagent must adapt to unfamiliar inputs | **Goals + quality criteria** (coverage, diversity, recency) |
| Subagent reports "couldn't do it" instead of trying alternatives | Same — the procedure had no failure branch |
| ❌ Not this | Exact steps; exact steps + fallbacks; branch-per-topic-type instruction sets |

### Context degradation remedy

| Condition | Choice |
|---|---|
| Specifics are being replaced by generalities, and new work is starting | **Summarize key findings → spawn a subagent seeded with that summary** |
| Exploration is broad and open-ended | Spawn subagents for bounded questions; main agent coordinates |
| ❌ Not this | `/clear` (loses discovered specifics); bigger model; more targeted prompts in the same crowded window |

---

## 1.4 Workflow enforcement and handoff

### Where to put a guarantee

| Mechanism | Bypassability | Use for |
|---|---|---|
| Logic **inside the tool** | Lowest | Business rules, money limits, authorization |
| `permissions.deny` | Low | Absolute path/tool prohibitions |
| `PreToolUse` / `PostToolUse` hook | Low | Interception; mechanical transformation |
| Check **around** the loop | Low | End-state guarantees |
| System prompt / CLAUDE.md / examples | Highest | Style, tone, preferences, conventions |

### Derived placement rules

| Condition | Placement |
|---|---|
| Rule is intrinsic to what the tool does | Inside the tool — and implement the policy (e.g. create a pending-approval record), not just an error |
| Rule requires intercepting a call to a tool you don't control | A blocking hook |
| Rule must hold when the process **terminates**, however it terminated | Code **outside** the loop |
| Any non-zero violation rate on an absolute rule | Move down the ladder — never rewrite the instruction |
| The control depends on the model picking the safe option | ❌ Choice is the vulnerability |
| The control is a parameter/flag the model sets | ❌ Bypassable by construction |

### Escalation triggers

| Trigger | Include? |
|---|---|
| Customer explicitly asks for a human | ✅ |
| Issue needs a policy exception beyond the agent's authorization | ✅ |
| Agent cannot make meaningful progress | ✅ |
| Rules engine mapping issue type → escalation | ❌ Removes judgment; misses novel cases |
| Sentiment / frustration score | ❌ A proxy, not the trigger |
| After N failed tool calls | ❌ Arbitrary; too early or too late |

### What a handoff must carry

| Include | Exclude |
|---|---|
| Customer identity and context | Only the original message (human redoes the work) |
| The records already retrieved | A database reference ID (adds a lookup) |
| The identified issue and why it needs a human | An attempted unauthorized action to "test" the limit |

### Error rate vs. consequence

| Option type | Preference |
|---|---|
| Lowers the error rate (confirm the match before acting) | ✅ Prefer |
| Softens the consequence (soft delete, undo window) | ❌ Mitigation, not a fix |
| Removes the usability that created the value | ❌ |
| Solves a different problem (de-duplication) | ❌ |

---

## 1.5 Agent SDK hooks

### Mechanism by requirement type

| Requirement type | Mechanism |
|---|---|
| Must never happen (path, tool, operation) | `permissions.deny` |
| Must always happen, mechanically, after an action | `PostToolUse` hook with a tool matcher |
| Must intercept and block before execution | `PreToolUse` hook |
| Should generally happen; judgment permitted | CLAUDE.md |

### Hook timing

| Hook | Fires | Can block? | Typical use |
|---|---|---|---|
| `PreToolUse` | Before the tool runs | **Yes** | Policy interception, redirection |
| `PostToolUse` | After the tool runs | No | Formatters, linters, tests, notifications |

### Reading a violation rate

| Observation | Conclusion |
|---|---|
| Prompt guidance yields a non-zero violation rate | The mechanism class is wrong |
| Adding emphasis **halved** the rate | Confirms it's probabilistic — more emphasis won't reach zero |
| A compliance rule is at 97% | That's a 3% failure; compliance is binary |

---

## 1.6 Task decomposition strategies

| Condition | Pattern |
|---|---|
| The same fixed steps run every time | **Prompt chaining** |
| The next step depends on what the last one found | **Dynamic / adaptive subtask generation** |
| Scope is open-ended and must be triaged | **Map → prioritize by impact → revise** |
| Subtasks are independent and known in advance | Parallel dispatch |
| Subtasks genuinely vary per run | Orchestrator-workers |

### The common over-engineering trap

| If the subtasks… | Then orchestrator-workers is… |
|---|---|
| Vary per run | Correct |
| Are identical every run | ❌ Over-engineering — there is nothing to orchestrate |

---

## 1.7 Session state and resumption

### Session selection *(the core table)*

| Situation | Choice |
|---|---|
| Continue a specific past investigation | `--resume <name>` |
| Continue the most recent thing | `--continue` |
| Some files changed since last time | Resume **and name the changed files** |
| Compare two approaches fairly from shared context | **Fork** |
| Old tool results are now wrong | **New session + structured summary + fresh tool calls** |
| A fact must be *current*, not historical | Structured state object re-sent every request |
| Crash recovery mid-pipeline | Structured per-agent export + coordinator manifest |

### Flag reference

| Flag | Loads |
|---|---|
| `--resume <name>` | A specific named session |
| `--continue` | The most recent session |
| `--session-id <uuid>` | A session by UUID (works, but requires tracking UUIDs) |

### Statelessness consequences

| Observation | Cause |
|---|---|
| Earlier turns appear forgotten **from the first test** | Prior messages not re-sent in the `messages` array |
| Superseded values still being referenced, resource usage low | Model inferring currency from chronology → hoist to a state object |
| Stale tool output persists **even after fresh calls** | The old `tool_result` is still in context competing — remove it, don't instruct around it |

| Proposed cause | Why it's wrong |
|---|---|
| "Memory is limited to N turns by default" | No such default; the bound is tokens |
| "The prompt needs an instruction to remember" | Recall comes from what's sent, not from being told |
| "A tool cleared the conversation state" | Tools don't touch the `messages` array |

---

# Domain 2 — Tool Design & MCP Integration

## 2.1 Tool interface design

### Symptom → structural fix

| Symptom | Fix |
|---|---|
| Model invents a parameter value | State **provenance** in the description ("must come from `<tool>`") |
| Model can't identify *which* entity (aliases, formats, duplicates) | Add a **lookup tool returning an identifier** |
| Downstream can't act on the result | Return **structured fields + stable IDs**, never prose |
| Parameters irrelevant or missing per operation | **Split the tool by operation** |
| Invalid parameter combinations are possible | Split so invalid combinations are **unrepresentable** |
| Output shape varies run to run | Replace a free-text instruction field with **purpose-specific tools** with defined contracts |
| Wrong one of two similar tools chosen | Expand **descriptions** to carry the distinction and the policy |
| Selection accuracy falls with a large catalog | Restrict per role; at large scale use **discovery-based availability** |
| Preview/confirm is bypassed | Preview tool issues a **single-use token**; execute tool requires it |
| "No longer available" between check and act | Make it **one atomic tool** |
| Huge result sets cause latency | First page + total count + **cursor** |
| Accuracy falls as total input approaches the limit | Tool schemas share the context budget — trim them |

### Split or merge

| Observation | Action |
|---|---|
| One tool, irrelevant/omitted parameters, invalid combinations | **Split** by operation |
| Two tools with genuinely overlapping purpose, wrong one chosen | **Merge**, or sharply differentiate descriptions |

### Validation vs. identification

| Ambiguity is about… | Fix |
|---|---|
| The **format** of a value | Enums, patterns, types |
| **Which entity** is meant | An identifier from a lookup — constraints can't separate two valid candidates |

### Result-set handling

| Approach | Verdict |
|---|---|
| First page + total count + cursor | ✅ Caller decides; truncation is visible |
| Auto-fetch all pages | ❌ Latency |
| Silent cap at N pages | ❌ Hides incompleteness — worse than slow |

---

## 2.2 Structured error responses

### Error classification

| Condition | Classification |
|---|---|
| Request malformed; required input missing; contract violated | **JSON-RPC protocol error** — the call itself is invalid |
| Call well-formed and executed; downstream returned not-found | **Tool result with `isError: true`** |
| Call well-formed and executed; downstream unavailable/timeout | **Tool result with `isError: true`** |

> The test is whether the **call** was valid — never how severe the failure sounds.

### Who handles the failure

| Failure type | Handler | Payload |
|---|---|---|
| Transient (timeout, unavailable, rate limit) | **The tool**, bounded retry with backoff | — |
| Permanent / business rule | **Surface to the model** | `retriable: false` + a customer-usable reason + what to try next |

### Minimum error payload

| Field | Purpose |
|---|---|
| `isError` / `is_error` | Preserve the error signal — never return a failure as normal content |
| `errorCategory` | transient / validation / permission / not_found |
| `isRetryable` | Makes the recovery decision mechanical |
| Message | What went wrong **and what to try next** |

### Diagnosing from behaviour

| Observed recovery behaviour | Cause |
|---|---|
| Inconsistent — sometimes retries, sometimes escalates, sometimes asks | Undifferentiated error payload |
| Retries something that will never succeed | Permanent errors indistinguishable from transient |
| Escalates immediately without trying alternatives | Message didn't say what else to try |
| Treats a failure as success | Error flag was dropped |

### Partial progress

| Condition | Response |
|---|---|
| Earlier steps succeeded, a later step failed | Report what was learned + name the failure + offer a path forward |
| ❌ Not this | Retry silently until success; escalate and discard the partial work; report only the failure |

---

## 2.3 Tool distribution and tool choice

### `tool_choice` values

| Value | Effect | Use when |
|---|---|---|
| `{"type": "auto"}` | Model decides whether to use a tool | Default conversational behaviour |
| `{"type": "any"}` | Must call **some** tool, its pick | You need structured output; the right tool depends on the input |
| `{"type": "tool", "name": "x"}` | Must call **tool x** | A specific step must happen first |
| `{"type": "none"}` | No tools | Pure text turns |

### Applying the constraint

| Condition | Choice |
|---|---|
| A prerequisite step must run first | Force the specific tool **for that turn only**, then release |
| Output must be structured but the type is unknown in advance | `"any"` — one call, no classification round trip |
| ❌ Not this | Forcing a specific tool on every call (later steps can never run) |

### Catalog size

| Catalog size | Strategy |
|---|---|
| A handful | Define them all |
| ~10–20 across several roles | **4–5 relevant tools per subagent** |
| 50+ | Discovery / deferred loading — hidden until surfaced |

### Diagnosing selection problems

| Observable | Cause |
|---|---|
| Confidently calls tools outside its role | **Selection accuracy** — too many candidates |
| Malformed or missing calls | Truncation |
| Reordering the tools array | ❌ Not a reliable priority signal |

---

## 2.4 MCP server integration

### The three primitives

| Primitive | Invoked by | Surfaced as |
|---|---|---|
| **Tools** | The model | Regular entries in the tool list |
| **Resources** | The application / browsed | Attachable contextual data |
| **Prompts** | **The user** | `/mcp__<server>__<prompt>` slash commands |

### Configuration scope

| Audience | Location |
|---|---|
| Team-shared, travels through version control | Project **`.mcp.json`** (committed) |
| Personal or experimental | **`~/.claude.json`** |

### Behaviour facts

| Question | Answer |
|---|---|
| How are tools from multiple servers made available? | **All configured servers' tools are discovered at connection and available simultaneously** |
| Does the agent select a server first? | No — no server-selection step, no routing by name prefix |
| Can one request combine tools from different servers? | Yes |

### Choosing the integration

| Condition | Choice |
|---|---|
| A purpose-built server already exists for the system | **Reuse it** |
| Nothing suitable exists | Build a custom server |
| ❌ Not this | Shelling out via Bash/curl (no schema, no discoverability); static exports (go stale) |

### When exploration is blind

| Condition | Fix |
|---|---|
| Many exploratory calls because the agent can't see what exists | Expose each server's **content catalog as resources** |
| A built-in tool keeps winning over your MCP tool | Expand the MCP tool's **description** with its unique capability and output |

---

## 2.5 Built-in tools

| Need | Tool |
|---|---|
| Find text **inside** files | **Grep** |
| Find files by **name or path** pattern | **Glob** |
| Load file content into context | **Read** |
| Targeted change with unique surrounding text | **Edit** |
| Create, or rewrite when no unique anchor exists | **Write** |
| Builds, tests, git | **Bash** (not file editing) |

| Condition | Choice |
|---|---|
| You have a distinctive literal string (error message, function call) | **Grep for it, then Read the hits** |
| `Edit`'s `old_string` can't be made unique | **Read → construct → Write** |
| Before any `Write` or `Edit` | **Read first** |
| ❌ Not this | A 30-line `old_string` to force uniqueness; Bash heredoc appends; reading directories systematically |

---

# Domain 3 — Claude Code Configuration & Workflows

## 3.1 CLAUDE.md hierarchy and scoping

### File locations

| Location | Scope | Reaches a teammate? |
|---|---|---|
| `~/.claude/CLAUDE.md` | User — all projects on your machine | ❌ Never |
| `<project-root>/CLAUDE.md` | Project | ✅ If committed |
| `<subdirectory>/CLAUDE.md` | That subtree | ✅ If committed |

### Behaviour facts

| Question | Answer |
|---|---|
| How are multiple CLAUDE.md files resolved? | Loaded as guidance and **combined** — not a strict override chain |
| Is it enforcement? | **No.** Absolutes belong in `permissions.deny` or a hook |
| How do you verify what loaded? | `/memory` |

### Modular organisation

| Condition | Mechanism |
|---|---|
| Compose a document from reusable pieces; per-location selection | `@path/to/file.md` import |
| A convention applies to a file type spread across directories | `.claude/rules/` with `paths:` |
| Shared standards duplicated everywhere | Each location imports **only** what it needs |
| ❌ Not this | One shared file everyone imports (duplication returns); prose override instructions |

### Quality of generated output

| Condition | Fix |
|---|---|
| Output quality is poor **and** added latency is forbidden | Document criteria, available fixtures, and contrasting examples → improve the **first pass** |
| ❌ Not this | Post-generation filtering, scoring passes, restricting where work happens |

### Review noise

| Noise source | Rule type |
|---|---|
| Already enforced by another tool | Skip |
| Generated / non-human-authored code | Skip |
| Intentional pattern that resembles a problem | **Require evidence** — cite a line showing actual incorrect behaviour |

> Put these in a discoverable, version-controlled file at the repo root rather than a workflow parameter.

---

## 3.2 Custom slash commands and skills

### Always-on vs. on-demand

| Condition | Mechanism | Context cost |
|---|---|---|
| Relevant to every session | CLAUDE.md | Every session |
| Relevant when touching a kind of file | `.claude/rules/` with `paths:` | Matching sessions |
| Relevant when doing a kind of task | Slash command / Skill | Only when invoked |
| Needed exactly once | The prompt itself | One turn |

### File shape decides whether it works

| Path | Shape | Creates |
|---|---|---|
| `.claude/skills/<name>/SKILL.md` | Directory + `SKILL.md` entrypoint | `/<name>` — canonical |
| `.claude/commands/<name>.md` | Flat file | `/<name>` — legacy alias |
| `.claude/skills/<name>.md` | Flat file in the skills dir | **Nothing** |

When a skill and a command share a name, **the skill wins**.

### Skill location

| Audience | Path |
|---|---|
| Project — committed, shared, stays in sync | `.claude/skills/<name>/SKILL.md` |
| Personal | `~/.claude/skills/<name>/SKILL.md` |

### Skill frontmatter

| Key | Guide (keyed answer) | Current behaviour |
|---|---|---|
| `context: fork` | Isolates verbose work in a sub-agent context | Same — no divergence |
| `allowed-tools` | **Restricts** the skill's tool access | **Pre-approves** listed tools; does not restrict |
| `argument-hint` | Prompts for required parameters | Autocomplete label only; never stops to ask |
| `description` | *Not on the guide's list* | What Claude matches against to auto-invoke — the field doing the most work |

Also live: `disallowed-tools`, `disable-model-invocation`, `model`, `effort`, `when_to_use`, `paths`.

| Question | Answer |
|---|---|
| Is `allowed-tools` a security boundary? | **No** — `disallowed-tools` or a deny rule is |
| A skill is never auto-invoked. Why? | Its `description` doesn't say *when* to use it |
| Verbose skill output polluting the conversation? | `context: fork` |
| Does a CLAUDE.md block create an invocable command? | No |

---

## 3.3 Path-specific rules

| Mechanism | Loads when | Scoped by |
|---|---|---|
| Root `CLAUDE.md` | Always | Nothing |
| Subdirectory `CLAUDE.md` | Working in that subtree | **Directory location** |
| `@<path>` line (there is no `@import` keyword) | Whenever the importing file loads | Not conditional; inlines eagerly, same token cost |
| `.claude/rules/` **without** frontmatter | Always | Nothing |
| `.claude/rules/` **with** `paths:` | Matching files are touched | **Glob pattern** |

| Condition | Choice |
|---|---|
| Token cost / context bloat from an always-loaded config | **Path rules** |
| The convention follows a **file type** across many directories | **Path rules** (globs); subdirectory files can't express this |
| The convention follows a **location** | Subdirectory `CLAUDE.md` |
| ❌ Not this | Reorganizing with headers (readability only); an `@<path>` line (inlines eagerly — same token cost) |

---

## 3.4 Plan mode vs. direct execution

> Decide on **ambiguity and blast radius**, not difficulty.

| Condition | Choice |
|---|---|
| One file, one function, mechanical | **Direct** |
| Evidence already localizes the problem (a stack trace, a specific error) | **Direct** — even in unfamiliar code |
| Many files, cross-module, breaking changes | **Plan first** |
| Multiple viable architectures with different trade-offs | **Plan first**, present options |
| You don't know which **code** is affected | **Plan mode** |
| You don't know what the **requirements** are (unfamiliar domain) | **Have Claude interview you** |

| Unknown | Remedy | Why not the other |
|---|---|---|
| Which code is affected | Plan mode explores what exists | — |
| What correct behaviour even is | Interview | Planning can't surface a requirement nobody recorded |

---

## 3.5 Iterative refinement techniques

| Condition | Technique |
|---|---|
| Complex behaviour with edge cases and performance requirements | **Write the test suite first**; iterate on failures |
| A specific known defect | **A concrete failing test case** |
| Must match existing conventions, **one-off** task | `@` references to the real files |
| Must match existing conventions, **recurring** | Document in CLAUDE.md |
| Several defects that **interact** | One at a time, test after each |
| Broad, ambiguous quality work | analyze → propose → implement → review |
| Mechanical rename or transformation | Just do it — phases add overhead with no quality gain |

### Batch or sequence feedback

| Relationship between changes | Handling |
|---|---|
| They interact (one affects another's outcome) | **Sequence** — one at a time, so cause and effect stay attributable |
| They're independent | Batch them together |

### Ranking of signals

| Rank | Signal |
|---|---|
| 1 | An automatically-checkable test |
| 2 | A concrete input/output example |
| 3 | A referenced real file |
| 4 | Prose description |

---

## 3.6 CI/CD integration

### Flag reference

**Non-interactive.** `-p` / `--print` — the single most directly tested fact in D3. Without it a CI job hangs. `CLAUDE_HEADLESS=true` and `--batch` **do not exist**.

| System prompt | Effect |
|---|---|
| `--system-prompt "<text>"` | **Replaces** the entire default prompt |
| `--system-prompt-file <path>` | **Replaces** it with a file |
| `--append-system-prompt "<text>"` | **Appends** text |
| `--append-system-prompt-file <path>` | **Appends** a file |

| Output & limits (print mode) | Effect |
|---|---|
| `--output-format text\|json\|stream-json` | Output shape |
| `--input-format text\|stream-json` | Input shape |
| `--json-schema '<schema>'` | Schema-validated output — lands in the envelope's **`structured_output`** field |
| `--max-turns <n>` | Cap agentic turns, then exit |

| Permissions, tools, context | Effect |
|---|---|
| `--permission-mode <mode>` | `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`, `manual` |
| `--allowedTools "<rules>"` | Runs **without a permission prompt**, e.g. `"Bash(git diff *)"` |
| `--disallowedTools "<rules>"` | Deny rules; a bare tool name **removes the tool from context** |
| `--tools "Bash,Edit,Read"` | Restrict which built-in tools exist at all |
| `--add-dir <path>` | Grants **file access**, not configuration discovery |
| `--model <alias\|name>` | Session model |

| Session & start-up | Effect |
|---|---|
| `-c` / `--continue` | Most recent conversation in this directory |
| `-r` / `--resume <id\|name>` | A specific session |
| `--bare` | Skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, CLAUDE.md. Leaves **Bash + file read/edit only** |

> `--max-budget-usd` appears in one bank question but **not** in the official flag reference. Use `--max-turns` as the reliable iteration cap.

### Append vs. replace

| Condition | Choice |
|---|---|
| You want custom instructions **plus** built-in tool-use behaviour | `--append-system-prompt` |
| You deliberately want to discard built-in behaviour | `--system-prompt` |
| Claude has the tools but doesn't use them appropriately | The default prompt was **replaced** — `--allowedTools` grants permission, not know-how |

### Cost and time control

| Requirement | Mechanism |
|---|---|
| Caps enforced **by Claude Code** | `--max-turns` + `--max-budget-usd` |
| ❌ Job-runner timeout | Kills the process; no graceful termination, no result |
| ❌ A cheaper model | Lowers unit cost; enforces no cap |
| ❌ Console monitoring | After the fact |

### Startup latency

| Condition | Choice |
|---|---|
| Auto-discovery is the measured cost, but project standards must still load | `--bare` **+** `--append-system-prompt-file <path>` |
| ❌ Not this | `--bare` + `--system-prompt-file` (loses built-in guidance); duplicating criteria into the CI script |

### Review hygiene

| Condition | Choice |
|---|---|
| Code was just generated by a session | Review in a **separate invocation** |

---

# Domain 4 — Prompt Engineering & Structured Output

## The domain triage table

| Symptom | Broken layer | Fix |
|---|---|---|
| Malformed JSON, prose around the object, parse failures | **Shape** | Tool use with an input schema |
| Schema-valid but invented / inconsistent / wrong granularity | **Judgment** | Few-shot examples; optional & nullable; enum escape hatches |
| Passes the schema but violates a business rule | **Meaning** | Semantic and cross-field validation |
| Correct early, degraded later | **Attention** | Reinforcement, versioning, splitting the work |

---

## 4.1 System prompts with explicit criteria

### Placement authority

| Placement | Authority |
|---|---|
| **System prompt** | Highest — sent every call, applies conversation-wide |
| First assistant message | Weaker — one turn among many |
| Prefixed to each user message | Weak — reads as user content |
| Environment variables | None — outside the API payload |

### What goes where

| Content | Placement |
|---|---|
| Tone, reasoning style, when to ask | System prompt |
| Changing runtime state | System prompt, refreshed before the next call |
| A response **opening** you want varied | A partial assistant prefix |

### Drift diagnosis

| Observation | Cause | Fix |
|---|---|---|
| Fails from the **first** attempt | History not being sent | Send prior turns |
| Correct early, **gradually** degrades, resource usage low | Accumulated turns competing with instructions | Reinforce at natural breakpoints (user-role messages) |
| Fails **abruptly at turn 2** | Would indicate "sent only once" — rarely the actual scenario | — |
| Old conversations contradict themselves after a prompt update | New behavioural contract applied to old history | **Version prompts**; bind each conversation to its version |

### Criteria design

| Condition | Choice |
|---|---|
| Enumerated triggers miss unanticipated signals | Replace with a **general principle**; keep only safety conditionals |
| The check has internal structure being skipped | "**Enumerate** every X, then **verify** each" |
| The gap is *process* | Instructions |
| The gap is *what the output should look like* | Examples |

### Ambiguity handling

| Condition | Behaviour |
|---|---|
| Reversible, unknowns are parameters | Infer → **state the assumption** → act → invite correction |
| Irreversible, or unknowns determine which action | One targeted question |
| ❌ Never | A battery of questions; a compound question; a silent default |

---

## 4.2 Few-shot prompting

### When examples are the answer

| Condition | Verdict |
|---|---|
| The failure is a **judgment call** (granularity, what counts as "stated", verbatim vs. normalized) | ✅ Examples |
| The convention is house style the model can't infer | ✅ Examples |
| Context-dependent acceptability must generalize to unseen cases | ✅ Examples |
| Verbose abstract guidance is losing attention | ✅ Examples — more concrete *and* more compact |
| Information is genuinely absent from the source | ❌ Schema change (optional/nullable) instead |
| Shape compliance is the problem | ❌ Tool-use schema instead |

### The four distractor families

| Family | Why it loses |
|---|---|
| Post-processing (regex, keyword filters, canonical mapping) | Repairs output; doesn't teach the judgment |
| Schema change (make required, add metadata enum) | Changes the container, not the decision |
| Model / parameter change (bigger tier, temperature 0, fine-tune) | Assumes a capability gap when the gap is instruction clarity |
| More prose (longer spec, "be conservative") | Abstract where the problem is concrete |

### Specific traps

| Proposal | Verdict |
|---|---|
| Fine-tune on accumulated corrections for a narrow pattern | ❌ Heavy machinery for something examples fix |
| Temperature 0 for consistency of convention | ❌ Consistently wrong, not correct |
| A numeric cap to control output size | ❌ Fixes one symptom of several |

---

## 4.3 Structured output with tool use

### Schema design checklist

| Need | Design |
|---|---|
| Guaranteed shape | **Tool use with an input schema** — enforced at generation time |
| A field that may be absent | Optional / nullable **plus** an explicit "return null if not stated" instruction |
| "Not stated" for a list | Empty array |
| "Stated but ambiguous" | An explicit `"unclear"` enum member |
| An open category set | Closed enum + `"other"` + a detail field |
| Values that change over time | Array of `{value, source, effective_date}` |
| Internal consistency | Separate `calculated_*` and `stated_*` fields; **flag** on mismatch |
| Format consistency | Normalization rules in the prompt, applied at extraction |

### Fabrication causes and non-causes

| Proposal | Effect on fabrication |
|---|---|
| Make fields optional/nullable + instruct `null` | ✅ Removes the pressure to invent |
| Add a confidence score | ❌ Annotates the invention; the model rates its own guess |
| Make fields required | ❌ **Increases** it — a required field is an instruction to produce a value |
| Stronger instructions alone | ❌ Can't override a schema demanding a value |
| Validation after generation | ❌ Detects, doesn't prevent; plausible inventions pass |

### Guaranteeing shape

| Approach | Verdict |
|---|---|
| Tool with an input schema matching the target | ✅ Constrained before parsing |
| Formatting instructions + a template in the prompt | ❌ Prose can still wrap or malform |
| Pre-filling an opening brace | ❌ Constrains the first character only |
| Regex extraction + retry | ❌ Post-hoc repair |
| Two calls — generate text, then format it | ❌ The second call is still unconstrained |

### Discrepancy handling

| Approach | Verdict |
|---|---|
| Extract both the derived and the stated value; **flag** mismatches | ✅ |
| A separate model to reconcile | ❌ Cost to guess which is right |
| Examples of consistent extractions | ❌ Can't fix a source-side error; flags nothing |
| Auto-adjust so the values agree | ❌ Manufactures agreement; destroys the signal |

---

## 4.4 Validation, retry, and feedback loops

### Is it retryable?

| Condition | Response |
|---|---|
| The correct value is **derivable** from what was supplied (format, structure, transformation) | Retry **with the specific error fed back** |
| The information **was never in the input** | ❌ Not retryable — fix the input |

| Example failure | Retryable? |
|---|---|
| Wrong date format, but the date was supplied | ✅ |
| Locale-formatted number | ✅ |
| Nested where flat was required | ✅ |
| A value that exists only in an unsupplied document | ❌ |

### What to include in the retry

| Failure type | Include |
|---|---|
| Syntax / structure | The failed output + the specific validation error |
| **Semantic** | The failed output + the error **+ the source document** |

### Non-fixes

| Proposal | Why it fails |
|---|---|
| Identical retry | Same input, same result |
| Temperature 0 | Makes the failure perfectly reproducible |
| Bigger model | Doesn't communicate the violated requirement |
| Stricter schema | Guards shape; can't enforce that values reconcile |

### The validation ladder

| Stage | Mechanism | Role |
|---|---|---|
| 1 | Tool-use schema | **Prevents** shape errors |
| 2 | Business-rule and cross-field checks | **Detects** semantic errors |
| 3 | Retry with the specific error + source | **Repairs** |
| 4 | Confidence-based routing to human review | **Contains** residual risk |

---

## 4.5 Batch processing strategies

| | Messages API | Message Batches API |
|---|---|---|
| Cost | Standard | **~50% discount** |
| Latency | Seconds | **Up to 24 hours** |
| Result identification | The response | **`custom_id`** per request |

### Routing

| Condition | Choice |
|---|---|
| Deadline comfortably exceeds the 24-hour window | **Batch** |
| Hard SLA shorter than the window | **Real-time** |
| Two populations with different deadlines | **Split** — batch one, real-time the other |
| Cost must drop but model and prompts must stay fixed | Batch (discount applies to all tokens) |

### Failure recovery

| Condition | Choice |
|---|---|
| A subset failed on input length | **Chunk only the failures and resubmit them**; merge partial results |
| A large corpus with a diverse failure tail | Submit all → analyze failures → resubmit **only failures** with refined prompts |
| ❌ Not this | Reprocess the whole batch; raise `max_tokens` for an input-length error; upgrade context size for everything |

### Why chunk

| Cause | Fix |
|---|---|
| Input exceeds the limit | Chunk → extract → merge |
| Both short and long inputs fit, but long ones are less accurate | **Attention dilution** — chunk anyway to focus each pass |
| ❌ Not this | Summarize first, then extract (discards the detail you're extracting) |

### Cost non-levers

| Proposal | Why it fails |
|---|---|
| Run requests concurrently | Reduces elapsed time, not unit price |
| Concatenate many items into one request | Still the real-time API; no discount; quality risk |
| Switch models when the model is fixed | Violates a stated constraint |

---

## 4.6 Multi-instance and multi-pass review

| Condition | Pattern |
|---|---|
| One pass must maximize coverage **and** suppress noise | **Finding pass** + separate **thresholding pass** |
| Strengthening one concern measurably weakens another | Separate focused prompts per concern, then merge |
| Output exceeds `max_tokens` | Split the input across calls; merge the result arrays |
| The reviewer also wrote the code | A **fresh, independent instance** |
| High-stakes fields needing certainty | Independent passes; reconcile disagreements |

### Diagnosing

| Observable | Cause |
|---|---|
| High precision, low recall, a real issue missed | Conflicting constraints in one pass |
| Improving A measurably degrades B | Overloaded prompt — attention budget is fixed |
| Output truncated mid-structure | Output ceiling |
| Self-review misses what an independent review catches | Anchoring on its own prior reasoning |

### Non-fixes

| Proposal | Why it fails |
|---|---|
| Raise `max_tokens`, shorten text | Buys headroom; next larger input truncates |
| Drop structured output for a plain list | Abandons the contract to solve a size problem |
| Add context (history, tests, dependencies) | More input, same conflicting instructions |
| Bigger model | Doesn't resolve structurally competing objectives |

### When merging passes

| Requirement |
|---|
| Deduplicate findings |
| Apply consistent severity rules |

---

# Domain 5 — Context Management & Reliability

## The information-kind table

| Kind of information | How it must survive |
|---|---|
| **Exact facts** — IDs, amounts, dates, statuses, statistics, named rules | Verbatim, in a structured block, re-sent every request, never summarized |
| **Conversational flow** — reasoning, discussion, rejected ideas | Progressively summarized; recent turns verbatim |
| **Provenance** — sources, dates, confidence, methodology | Carried in a separate channel through every stage |

---

## 5.1 Context window management

### Strategy selection

| Condition | Strategy |
|---|---|
| Long session; earlier conclusions are being asked about | **Hybrid** — progressive summarization of old blocks + recent turns verbatim |
| Rules/constraints must hold for the whole session | **Persistent block at context start, exempt from summarization** |
| Follow-ups need exact numbers and identifiers | **Structured fact store**, injected when precision is needed |
| Long exploration with repeated re-explaining | **Persistent scratchpad** of findings (not file contents) |
| Multiple independent threads in one session | **Per-thread structured facts layer**, independent of history |
| Repeated verbose tool responses | **Trim to task-relevant fields — before the next calls** |

### Why each common alternative fails

| Approach | Failure |
|---|---|
| Rolling-window truncation | Discards the older conclusions being asked about |
| A larger window | Delays, doesn't solve |
| One comprehensive summary | Destroys exact values, defined terms, identifiers |
| Vector retrieval instead of history | Latency; can't guarantee presence every turn; retrieves plausible-but-wrong neighbours |
| Prose summaries of structured data | Loses identifiers and precise status values |
| Clearing context | Discards what you learned along with the noise |

### Positional handling

| Condition | Action |
|---|---|
| Aggregating many sources | Key findings **at the start**; detailed sources under clear headers |
| Want the model to prioritize something | **Change placement** — instructing it to "pay attention" is weaker |

---

## 5.2 Escalation and ambiguity resolution

| Condition | Behaviour |
|---|---|
| Action is reversible and the unknowns are **parameters** | Infer → **state the assumption** → act → invite correction |
| Action is irreversible | Ask one targeted question |
| The unknowns determine **which action entirely** | Ask one targeted question |
| ❌ Never | Multiple questions; a compound question; a silent default |

| Distinction | Venue-style (parameters) | Action-choice style |
|---|---|---|
| What's missing | Parameters of one action | Which action |
| Can you assume? | Yes | No |
| Right move | Assume + state + invite | One targeted question |

---

## 5.3 Error propagation in multi-agent systems

### Structured error context — four required elements

| Element | Contents |
|---|---|
| **Failure type** | `transient` · `validation` · `business` · `permission` |
| **What was attempted** | Tool, query, parameters, target system |
| **Partial results** | Whatever was retrieved before the failure |
| **Alternative approaches** | The subagent's own suggested recovery paths |

### The two anti-patterns

| Anti-pattern | What it looks like | Consequence |
|---|---|---|
| **Silent suppression** (worst) | `{"results": [], "status": "success"}` after a timeout | Coordinator never retries; output looks complete but has invisible gaps |
| **Workflow termination** | One subagent fails → whole pipeline dies | Discards the other subagents' completed work |

**Correct middle ground:** the subagent reports structured context, the coordinator assesses, the system continues with partial results or targeted recovery.

### Access failure vs. valid empty result

| | Access failure | Valid empty result |
|---|---|---|
| Query executed? | **No** — timeout, connection error, permission denial | **Yes** |
| Is it the answer? | No | **Yes** |
| Retry? | Consider it | **No** |

| Conflation | Consequence |
|---|---|
| Access failure treated as empty result | Never retry when you should (= silent suppression) |
| Empty result treated as access failure | Waste retries on a query that will always return nothing |

### Other rules

| Rule |
|---|
| Subagents handle their **own** transient failures locally; propagate only what they can't resolve |
| Always include what was attempted and any partial results when propagating |
| Add **coverage annotations** to synthesis output — name the gaps, don't silently omit topics |
| A generic "unavailable" status hides query, partial results, and alternatives |

---

## 5.4 Codebase exploration

| Condition | Strategy |
|---|---|
| Finding all callers, and re-exports/aliases may exist | **Read the modules to enumerate every exposed name, then grep each** |
| Logic spans many files | **Find the base class/interface, read it first**, then trace implementations |
| Exploration is degrading the main context | Delegate bounded questions to subagents |

| Approach | Failure |
|---|---|
| Grep the original name only | Confidently incomplete |
| Consult documentation | Stale; misses internal usage |
| Read every importing file | Wasteful; still misses dynamic/qualified calls |
| Read the largest files first | File size ≠ architectural importance |
| Keyword-match without reading the design | Misses lifecycle and guarantees |
| Read everything | Burns the budget before producing value |

---

## 5.5 Human review and confidence calibration

### Routing

| Signal | Use for routing? |
|---|---|
| Low model confidence | ✅ |
| Conflicting evidence in the source (comparison tables, appendices, ambiguity) | ✅ |
| Entity type / field importance | ❌ A proxy for importance, not for risk |
| Pure randomness, across everything | ❌ Wastes scarce capacity |
| Downstream failure reports | ❌ Reactive; damage already done |

### Measurement

| Condition | Action |
|---|---|
| A confidence threshold creates an un-inspected tier | **Stratified random sampling** of that tier, continuously |
| About to reduce oversight based on a metric | **Disaggregate by segment first** — aggregates conceal floors |
| Considering double-extraction as an error check | ❌ Correlated errors survive; agreement ≠ accuracy |
| Considering heuristics for known failure modes | ❌ Can't surface new patterns |

### Tool output for review decisions

| Component | Purpose |
|---|---|
| Per-field values **with** raw confidence | Preserved for analysis and threshold tuning |
| A computed `requires_review` boolean | Applies tested thresholds — deterministic routing |
| A `review_reasons` array | Which field, and why — enables investigation and tuning |

| Observable | Cause |
|---|---|
| Both under-review **and** over-review at once | The model is interpreting raw scores against an unstated threshold |

---

## 5.6 Information provenance and multi-source synthesis

| Condition | Requirement |
|---|---|
| A stage compresses its input | That's where metadata dies — carry it in a **separate channel** |
| Claims need attribution downstream | **Structured claim→source mappings**, preserved and merged at every stage |
| Observations differ and were collected at different times | **Dates in the structured output** — without them a trend reads as a contradiction |
| Observations differ and used different methodologies | **Separate sections** for established vs. contested; explain *why* they differ |
| Different sources have different native formats | **Render each in its native format** — tables, prose, structured lists |

### Context to pass to a final consumer

| Option | Verdict |
|---|---|
| The condensed draft **+ a structured source index** | ✅ Exactly what's needed |
| Draft only, match claims to sources afterwards | ❌ Post-hoc similarity misattributes |
| Full accumulated context from every stage | ❌ Reintroduces the bloat the pipeline removes |
| A summary naming sources by name only | ❌ No URLs/locators; unverifiable |

### Combining conflicting estimates

| Approach | Verdict |
|---|---|
| Present both with their original characterization; explain the difference | ✅ |
| Weight-average by normalized confidence | ❌ False precision from incomparable methods |
| Filter out low-confidence findings | ❌ Manufactures unanimity |
| Require two-source corroboration | ❌ Discards unique findings |
| Treat the newest as authoritative | ❌ Discards the trend and the methodological difference |

---

# Cross-domain tables

## Guarantee placement (appears in D1, D2, D3, D4, D5)

| Requirement strength | Mechanism |
|---|---|
| Must never happen | Logic inside the tool; permission denial |
| Must always happen, mechanically | Deterministic hook |
| Must be true when the process ends | A check outside the process |
| Should generally happen; judgment allowed | Prompt-level guidance |
| Relevant only sometimes | Conditionally loaded guidance |

## Scope by audience and frequency

| Property | Location |
|---|---|
| Team-shared, must stay in sync | Committed project file |
| Personal / experimental | User-level config |
| Every session | Always-loaded guidance |
| Certain file types | Pattern-scoped rules |
| Certain tasks | Skill or slash command |
| Once | The prompt |

## Symptom → cause

| Signature | Cause | Fix class |
|---|---|---|
| Received **nothing** | Missing hand-off | Place the data explicitly |
| Present but **partial** | Capacity | Reduce or reference |
| Present but **imprecise** | Attention dilution | Re-assert, relocate, isolate |
| **Structurally broken** | Output ceiling | Split the work |
| Well-formed but **wrong** | Meaning unchecked | Semantic validation |
| Succeeds then fails on repeat | State changed between calls | Make it atomic |
| Narrates an action it never took | Capability unavailable | Grant/expose it |

## Split when one thing carries two objectives

| Tension | Split into |
|---|---|
| Coverage vs. noise suppression | Finding pass + thresholding pass |
| Several concerns in one prompt | One focused pass per concern, then merge |
| Author and reviewer | Independent instances |
| Comparing two options from shared context | Two branches from the shared point |
| Input too long | Chunks |
| Output too long | Multiple calls |

## What never to do with data

| Action | Instead |
|---|---|
| Silently reconcile a discrepancy | Flag it for review |
| Discard the older of two conflicting observations | Attach dates and present the sequence |
| Average incomparable estimates | Present each with its characterization |
| Filter out disagreement | Separate established from contested |
| Replace structured values with prose | Keep the structure |
| Flatten every format to one intermediate | Preserve native representations |

---

See also: [Overview](00-Overview-and-Priorities.md) · [Pattern Decoder](01-Pattern-Decoder.md) · [D1](Domain-1-Agentic-Architecture-Orchestration/summary.md) · [D2](Domain-2-Tool-Design-MCP-Integration/summary.md) · [D3](Domain-3-Claude-Code-Configuration-Workflows/summary.md) · [D4](Domain-4-Prompt-Engineering-Structured-Output/summary.md) · [D5](Domain-5-Context-Management-Reliability/summary.md)
