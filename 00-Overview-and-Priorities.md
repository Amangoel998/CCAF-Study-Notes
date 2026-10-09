# CCA-F — Overview, Cross-Cutting Principles, and Study Priorities

**Exam format: 60 questions · 120 minutes · five weighted domains.**

These notes are derived from the categorized bank in `categorized_exam_dump/` — **236 questions** (175 from the PDF dump + 61 from screenshots), checked for answer/explanation consistency and sorted into the blueprint's 30 subtopics. Each domain file explains the subtopics with the scenarios they're tested through, the reasoning behind the correct answer, and why each attractive distractor fails.

**Companion files:** [01-Pattern-Decoder.md](01-Pattern-Decoder.md) for domain-independent decision rules · [02-Decision-Tables.md](02-Decision-Tables.md) for every lookup table in one place · each domain's `N.0-cross-topic-and-domain-level.md` in the dump for authored cross-subtopic questions.

---

## Coverage vs. blueprint

| Domain | Blueprint | Bank | Distinct concepts | What that means for you |
|---|---|---|---|---|
| D1 — Agentic Architecture & Orchestration | 27% | 60 (25%) | ~40 | Matches. The largest single domain; study broadly |
| D2 — Tool Design & MCP Integration | 18% | 53 (22%) | ~30 | Over-represented; highest question density per blueprint point |
| D3 — Claude Code Configuration & Workflows | 20% | 30 (13%) | **~18** | **Biggest gap**, and worse than the count implies — highest duplication rate in the bank |
| D4 — Prompt Engineering & Structured Output | 20% | 57 (24%) | ~33 | Over-represented by question count; nearly every question has a reworded twin |
| D5 — Context Management & Reliability | 15% | 36 (15%) | ~26 | Matches. But 5.3 has **zero** questions |

> **Read the "distinct concepts" column, not the question count.** The bank duplicates heavily: 236 questions cover roughly 150 concepts. A subtopic with 17 questions and 9 concepts is less covered than it looks. Each domain's `N.0` file lists the duplicate clusters.

### Two corrections already applied

1. An earlier version put D4 at ~36% and D2 at ~13%. That was a categorization artifact — many tool-design and error-response questions had been filed under D4.
2. The bank was re-audited question by question against what each **correct answer** tests. Ten questions were reassigned across subtopics and domains, because the original classifier matched surface keywords in the stem. The affected concepts are cross-referenced in both files they could plausibly belong to, since **the exam will not tell you which subtopic a scenario belongs to.**

---

## How the exam asks questions

Almost every question is a **scenario with a diagnosis hidden in it**. You are given a system, a symptom, often a number, and four plausible fixes. Three are real engineering practices that are wrong *for this symptom*.

The skill being tested is not "do you know what a hook is" — it is **can you read the symptom and identify which layer is broken**.

### Incidental detail is never incidental

Any fact a scenario states that isn't needed to understand the situation is there to eliminate options. Read each one as a constraint on the answer, not as colour.

| Kind of detail stated | What it eliminates |
|---|---|
| A resource is well below its limit | Every explanation and fix based on that resource's capacity |
| A fix was already attempted and **partly** worked | That entire fix class — partial success against a binary requirement is failure |
| A cost is forbidden (latency, complexity, extra passes, extra components) | Every option incurring it, however sound |
| A component must stay fixed (same model, same prompts, same quality) | Every option that changes it |
| An enforcement location is specified | Every option enforcing somewhere else, even if functionally similar |
| Two populations have different requirements | Any uniform treatment of both |
| Something keeps changing or growing | Any fixed enumeration of it |
| The work is one-off, or conversely recurring and shared | The wrong scope of configuration |
| A metric is given **in aggregate** | Trusting it before disaggregating |
| Several distinct symptoms are listed | Every option that resolves only some of them |

The full set of these axes, with the reasoning behind each, is in [01-Pattern-Decoder.md](01-Pattern-Decoder.md).

### Four failure signatures worth memorizing

The exam deliberately offers these as interchangeable explanations for one symptom. They are not interchangeable — each produces a different observable signature.

| Signature | Root cause | Fix class |
|---|---|---|
| **Total absence** — a step reports it received nothing at all | A hand-off was never made | Place the data in the downstream input explicitly |
| **Partial / thinned** — present but incomplete, early content dropped | Capacity overflow | Reduce what's sent; compress the compressible part |
| **Imprecise** — specifics replaced by generalities, instructions softened over time | Attention dilution from accumulated content | Re-assert, relocate, or isolate what must dominate |
| **Structurally broken** — cut off mid-structure, unparseable | Output ceiling | Split the work so each response stays under it |

The one that matters most: **a capacity explanation predicts partial output, never total absence.** Any option attributing "it got nothing" to a size limit is wrong by construction.

---

## The eight ideas that decide most questions

### 1. Nothing is shared implicitly

The Messages API is stateless. Subagents have no shared memory, no access to each other's transcripts, and no automatic retrieval of anything. If a later step needs earlier data, **your code must put it there**.

*Appears as:* synthesis agent reports "no research findings were provided" (D1); the agent re-asks a verification question already answered (D1, D4); a subagent "automatically accesses the shared memory store" (wrong — access must be configured).

*Trap answers:* "increase the context window", "share an API connection", "give the agent a tool to fetch the other agent's history."

### 2. Prompts advise; code enforces

Rank mechanisms by how hard they are to bypass:

```
tool-side logic  >  permissions.deny  >  hooks  >  orchestration checks  >>  prompts
```

When a requirement is absolute — a money limit, a protected directory, a guaranteed escalation — the answer is a hard mechanism. **Never stronger wording.** A 3% violation rate on a compliance rule is a finding, not a near-miss.

*Appears as:* the $500 refund/reimbursement scenarios (D1); the Prettier formatting hook (D1, D3); `max_turns` exhaustion needing a post-loop check (D1).

### 3. Fix the interface, not the wording

Fabricated parameters, wrong tool selection, unusable tool output, and uninterpretable errors are all **schema and description** problems. The model sees the schema and description at decision time; it never sees your validation code, and your system prompt doesn't travel with the tool.

Prefer **structurally impossible** over **discouraged**: split the tool, require a single-use token, hide tools until discovered.

*Appears as:* fabricated `order_id` (D2); `log_workout` accepting miles for bench press (D2); `dry_run=false` bypass (D2).

### 4. Structure ≠ correctness

A schema guarantees shape. It does not guarantee meaning. "Passed schema validation" and "is correct" are tested as completely separate concerns.

Corollaries:
- **Required fields cause fabrication.** Give the model a legal way to say "not stated" (optional/null, empty array) and "ambiguous" (an `"unclear"` enum value).
- **Confidence fields never prevent fabrication** — the model just rates its own guess.
- **Cross-field checks surface discrepancies; they must never silently correct them.**

*Appears as:* `attendee_count: 500` invented from a silent source (D4); the `calculated_total` vs. `stated_total` invoice check (D3, D4).

### 5. Match the recovery to the error

| Error type | Response |
|---|---|
| Transient (503, timeout, rate limit) | Retry **inside the tool** with bounded backoff |
| Permanent / business (already refunded, outside window) | Surface to the model with a category, a reason, and what to try next |
| Protocol (malformed request, missing required param) | JSON-RPC protocol error |
| Required data absent from the input | **Retrying cannot help.** Fix the input |

*Appears as:* uniform `"Operation failed"` errors causing erratic recovery (D2); retry-with-feedback loops (D4); the "least retryable" question (D4).

### 6. Preserve facts, compress prose

Context is a budget. Keep exact values — IDs, amounts, dates, statuses, statistics, named rules — in a **structured block re-sent every request** that summarization never touches. Summarize the conversation around it; keep recent turns verbatim.

**Prose is lossy for data.** Whenever an option replaces structured values with a narrative summary, it's wrong.

*Appears as:* the persistent case-facts block (D5); the structured state object for changing preferences (D1); tool-result trimming before further lookups (D5).

*Trap answers:* a bigger context window; a vector database for facts that must be present every turn.

### 7. Criteria, not procedures — and concrete, not descriptive

Two halves of one idea:

- **Tell a subagent what good output looks like, not the exact steps.** Prescribed procedures have no branch for "the procedure didn't work."
- **When you need the model to learn a judgment call, show it.** A failing test case, an `@`-referenced file, or a few annotated examples beats any amount of prose.

*Appears as:* detailed search instructions causing "insufficient results" (D1); few-shot examples for compound phrases and informal measurements (D4); `@` references instead of describing conventions (D3).

*Trap answers:* "add more conditional branches"; "write a comprehensive specification"; "fine-tune on the corrections" for a pattern a handful of examples fixes.

### 8. Separate things that pull against each other

When one pass is asked to do two opposing jobs, split it:

| Tension | Split |
|---|---|
| Maximize recall **and** suppress noise | Finding pass + thresholding pass |
| Cover security **and** API design **and** business logic | Separate focused prompts, then merge |
| Write the code **and** review it | Fresh, independent instance |
| Compare two approaches from shared context | **Fork** the session, don't explore sequentially |

The shared mechanism is independence: a single context that holds both objectives (or both the code and its justification) will favour one.

*Appears as:* the conservative-review prompt that missed a race condition (D4); same-session review vs. CI review (D4); fork vs. sequential exploration (D1).

---

## Cross-domain echoes

The same idea is tested in several domains under different names. Recognizing the echo saves time:

| Idea | D1 | D2 | D3 | D4 | D5 |
|---|---|---|---|---|---|
| Hoist current state out of the conversation | Structured state object | — | — | State in the system prompt | Persistent facts block |
| Review must be independent | Fork for comparison | — | Review in a separate invocation | Fresh-instance review | — |
| Surface, don't silently correct | — | Structured error payloads | `calculated_total` flagging | `calculated_total` flagging | Explicit uncertainty sections |
| Hard rules beat wording | Hooks, tool-side limits | Single-use tokens | `permissions.deny` | Tool-use schema | Computed `requires_review` |
| Split when overloaded | Prompt chaining vs. orchestrator | Split a tool by operation | Path-scoped rules | Multi-pass review | Chunk → extract → merge |
| Chunk/compress by cause | Subagents for focused context | Pagination with a cursor | — | Chunking long transcripts | Tool-result trimming |

---

## Study order

1. **D3 first.** 20% of the exam, thinnest coverage in the bank. Work through the official lessons: CLAUDE.md scoping, path rules, skills, plan mode, CI flags. The CI flag table is pure recall and the cheapest points on the exam.
2. **D1 context passing and session mechanics.** The densest trap family on the whole exam (absence vs. degradation) plus the resume/continue/fork table.
3. **D2 interface and error design.** Highest question density in the bank; learn the split/lookup/token patterns and the protocol-vs-result error boundary.
4. **D4 structured output.** The shape → judgment → meaning pipeline, and the optional-vs-required fabrication rule.
5. **D5 context and provenance.** Persistent facts, trimming, segment validation, dates on every finding.

Then re-read the **"exam traps"** section at the end of each subtopic — they're the distilled failure modes, and they're what you'll actually be tested on.

For the domain-independent decision rules — the ones you can apply to a scenario whose vocabulary you've never seen — work from [01-Pattern-Decoder.md](01-Pattern-Decoder.md).

---

## Sources

- [Curriculum](https://claudecertificationguide.com/learn) · [Blueprint](https://claudecertificationguide.com/architect-foundations) · [Quick references](https://claudecertificationguide.com/learn/quick-reference)
- [Anthropic documentation](https://docs.anthropic.com/) · [MCP specification](https://github.com/modelcontextprotocol)
