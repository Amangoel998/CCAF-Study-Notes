# Domain 5 — Context Management & Reliability

**Blueprint weight: 15% (~9 of 60 questions) · 36 questions in the bank, ~26 distinct concepts**

> Two questions were re-filed into this domain: one about a tool schema competing for the token budget (previously 2.1) and one about conversation history not being re-sent (previously 4.1). Both read like other domains' problems and are context-management concepts. See [5.0 Cross-Topic & Domain-Level](../../categorized_exam_dump/domain-5-context-management-reliability/5.0-cross-topic-and-domain-level.md).

> **The one sentence that governs this domain:** context is a budget — protect the few things that must stay **exact**, compress everything else, and attach **provenance** so later steps can tell a trend from a contradiction.

Domain 5 is about what survives. Three things are *not* interchangeable, and nearly every question hinges on keeping them apart:

| Kind of information | How it must survive | Why |
|---|---|---|
| **Exact facts** — IDs, amounts, dates, statuses, statistics, named rules | **Verbatim, in a structured block, re-sent every request** | Prose summarization destroys precision. "They looked at adults 18–65" is not "N=1,247, mean age 42.3, SD 11.8" |
| **Conversational flow** — reasoning, discussion, rejected ideas | **Summarized progressively**, recent turns verbatim | Needed for coherence, not for precision |
| **Provenance** — source URLs, publication dates, confidence, methodology | **Carried separately from content, through every stage** | Without it, a trend looks like a contradiction and a claim loses its citation |

A second rule applies throughout: **prose is lossy for data.** Whenever an option proposes replacing structured values with a narrative summary, it is almost certainly wrong.

---

## 5.1 Context window management

The densest subtopic (14 questions). Every scenario is a variant of: *a long session, the model "forgets" something, four strategies are proposed.*

### The baseline strategy: hybrid summarization

> Extended conversations reach **60K–85K tokens**. Users repeatedly ask about earlier conclusions, specific numbers (sample sizes, p-values), or established facts (allergies, measurements, themes) — and the assistant either forgets or hedges. The current strategy is a sliding window keeping the most recent 25–30 turns, or uniform summarization after 8 turns.

**Fix: a hybrid — progressively summarize older material while keeping recent exchanges verbatim.** Older blocks are replaced with concise summaries that **explicitly extract and preserve conclusions, decisions, key facts, and recurring themes**; the last ~5–10 turns (or last 20–30K tokens) stay word-for-word so the conversation still reads coherently.

```
┌─────────────────────────────────────────────────┐
│ System prompt                                   │
├─────────────────────────────────────────────────┤
│ PERSISTENT FACTS BLOCK  (never summarized)      │ ← exact values live here
├─────────────────────────────────────────────────┤
│ Summary of turns 1–30   (conclusions preserved) │ ← compressed prose
├─────────────────────────────────────────────────┤
│ Turns 31–40 verbatim                            │ ← coherence
└─────────────────────────────────────────────────┘
```

Why each competing strategy fails — this distractor set recurs across the whole subtopic:

| Strategy | Why it fails |
|---|---|
| **Pure rolling-window truncation** (keep the last 25–30K tokens) | Discards exactly the older conclusions users are asking about. Makes no distinction between ephemeral brainstorming and durable facts |
| **A bigger sliding window** (50 pairs instead of 25) | Delays the problem without solving it. The larger window fills and forgets too |
| **One comprehensive summary** of the entire history | Compresses away exact numbers, user-defined terms ("room temperature butter = 68°F"), and identifiers. **Prose is lossy for precise data** |
| **Semantic search / vector retrieval instead of history** | Adds latency, fragments context, and can retrieve passages that are semantically similar but factually different — a competitor's figure from a comparison table instead of the product's |

**Memorize the vector-database trap.** It appears in nearly every 5.1 question and it is almost never the answer. It adds latency, cannot guarantee the needed fact is in context on a given turn, and risks retrieving plausible-but-wrong neighbours.

### Persistent facts must be exempt from summarization

> Long creative/analytical sessions (**40+ turns**) must keep persistent constraints consistent — story bible elements (character traits, world rules, plot structure) or technical constraints (allergies, user-defined terms, architectural decisions). Meanwhile ephemeral content (brainstorming, rejected ideas, tangents) is consuming context.

**Fix: separate the persistent elements into a retained reference section positioned at the start of context, immediately after the system prompt. It is never summarized, truncated, or modified except by explicit user edit.** Progressive summarization and trimming apply only to the ephemeral discussion.

The block is a **read-only source of truth**. Two properties matter:

1. **Position** — at the start of context, which is also the lost-in-the-middle mitigation.
2. **Exemption** — summarization passes skip it entirely.

- *Pure sliding window* — loses the world rules; the model starts inferring them from recent discussion and contradicts itself.
- *Store them in a vector database and retrieve semantically* — cannot guarantee the rules are present **every** turn, which is the whole requirement.
- *Summarize everything every 20 turns* — treats persistent and ephemeral content identically, compressing world rules alongside dismissed ideas and garbling both.

### Precision questions need a structured fact store

> A research assistant works over **60K+ tokens**. Follow-up questions demand exact values: sample sizes (`n=1,247`), statistics (`p=0.032, 95% CI 18–24%`), methodological specifics (inclusion criteria, measurement protocols), and metadata (paper titles, publication years).

**Fix: maintain a structured JSON/database store of key facts extracted from each source, and inject the relevant entries when a precision-dependent question appears.** Keep raw methodology and results sections in context; summarize only the conversational interpretation around them.

```json
{"source_id": "smith2023",
 "n": 1247, "mean_age": 42.3, "sd": 11.8, "female_pct": 68,
 "p_value": 0.032, "ci_95": [18, 24],
 "inclusion": "adults 18-65, no prior diagnosis"}
```

The exam spells out the contrast directly: *"they looked at adults aged 18–65"* versus *"sample: N=1,247, mean age=42.3, SD=11.8, 68% female."* A prose summary of methodology is **inherently lossy**, and the loss is exactly the part follow-up questions need.

- *General RAG over summarized history* — you're retrieving from summaries that already destroyed the numbers.
- *A separate Claude call to restore fidelity* — generates a **new** summary rather than retrieving the original values. You cannot regenerate precision that was discarded.

### A scratchpad for exploration sessions

> Extended codebase exploration (**30+ minutes**). The agent gives inconsistent answers about modules discussed earlier; engineers have to keep re-explaining context for modules already explored.

**Fix: maintain a persistent scratchpad recording key findings from each exploration** — identified modules, their purposes, critical dependencies, API signatures, constraints and gotchas — and consult the scratchpad for later questions instead of re-reading the same files.

- *Upgrade to a model with a bigger window* — more room, same decay. Newly explored detail still competes with earlier findings.
- *Clear context every 15 minutes* — removes the contamination along with everything you learned.
- *Summarize all source files upfront* — compresses code into prose, losing the exact signatures and dependency edges needed to answer "what calls this?"

**Note: the scratchpad records *findings*, not file contents.** It's a distilled index, not a compression of the source.

### Structured layers for multi-issue sessions

> A customer support session covers three separate issues across **45 turns**: a refund (turns 1–15), a subscription question (16–30), a payment method update (31–45). At **turn 48** the customer asks *"What happened with my refund?"* The conversation is approaching the context limit.

**Fix: maintain a separate structured layer of concrete facts per issue — order IDs, amounts, status values, dates, action items — updated as information arrives but independent of the conversational history.** Answer the follow-up from the layer, not by searching the transcript.

```json
{"issues": [
  {"type": "refund",  "order_id": "ORD-8841", "amount": 129.99,
   "status": "approved", "date": "2024-05-02", "next_action": "funds in 3-5 days"},
  {"type": "subscription", "plan": "Pro", "status": "paused", "resume_date": "2024-06-01"}
]}
```

- *Sliding window of the last 30 turns* — at turn 48, turns 1–15 are gone. The refund details are simply unavailable.
- *A narrative summary of earlier turns* — loses the exact order ID and, critically, the **precise status**. "Approved" vs. "pending" vs. "cancelled" is a one-word difference with completely different consequences, and prose summarization blurs exactly this kind of distinction.
- *Re-fetch on demand via MCP tools* — adds latency and requires the user to identify the issue precisely enough to look up. It fails on "that refund I asked about earlier."

### Trim tool results before they accumulate

> An agent repeatedly calls `lookup_order()` while investigating returns. Each response carries **40+ fields**: items, shipping details, payment info, status history, tracking, variant options. After several calls the tool responses dominate the context. The customer now mentions **two more orders** to discuss.

**Fix: extract only the fields the task actually needs** — items, purchase date, return window status, reason for return — and drop the verbose metadata (carrier codes, payment processor IDs, full variant hierarchies). **Do this before making additional lookups**, so the space is reclaimed in advance.

The timing matters. Trimming after the next two lookups is too late; the context is already full.

- *Proceed without trimming* — you run out before covering three orders.
- *Move everything to a vector database with semantic indexing* — latency, and you still have to inject the relevant fields back into context to reason over them.
- *Replace the structured responses with prose summaries* — loses exact order IDs, dates, and status codes. The scenario names the specific risk: confusing "pending return" with "return in transit."

**Upstream version of the same idea:** ask subagents to return *structured findings with citations, dates, and relevance metadata* rather than verbose reasoning or raw content. Cheaper, and better provenance.

### Lost in the middle

Models attend most reliably to the **beginning** and **end** of a long input. When aggregating many sources, put a concise **key findings** section at the top and separate detailed source outputs under clear headers.

**The fix is structural, not prompt-based. Changing placement beats instructing the model to "pay attention to this."**

```markdown
## Key Findings Summary
- Source A: 12% market growth in renewable sector (2023)
- Source B: Patent filings increased 34% year-on-year
- Source C: Regulatory framework delayed until Q3 2025

## Detailed Findings
### Source A: Market Analysis Report
...
```

### Where trimming should happen

Tool-result trimming belongs in a **`PostToolUse` hook or in the tool implementation itself** — *before* the result enters the conversation history. **Once verbose data is in context it stays there for every subsequent turn.** (This is the 1.5 normalisation hook doing double duty.)

### Truncation is not a strategy

The API is stateless, so every request must carry the complete history for coherence — in direct tension with the token budget. **Selectively truncating history breaks conversational coherence.**

The persistent case-facts block is what resolves the tension: it separates the critical facts from the summarisable narrative, so you can compress the conversation flow while preserving every transactional detail.

> **Trap:** believing conversation history can be selectively truncated without consequences. Use case-facts blocks plus summarisation — never truncation.

### Upstream agent optimisation

Upstream agents often return verbose reasoning chains the downstream agent cannot use. **Have them return structured data — key facts, citations, relevance scores, dates — instead of prose.**

```json
{"findings": [{"claim": "Renewable energy investment grew 12% in 2023",
               "source": "IEA World Energy Report 2024",
               "sourceUrl": "https://example.com/report",
               "relevanceScore": 0.92,
               "publicationDate": "2024-01-15"}]}
```

Tokens aren't the only win: structured upstream output lets downstream agents process findings **without re-parsing prose**, and it carries the provenance 5.6 depends on.

### Prompt caching — explicitly out of scope

Place stable content (system instructions, tool definitions, reference documents) **before** volatile content and mark the boundary with a `cache_control` breakpoint. Caching matches prefix-by-prefix from the start of the prompt, so putting dynamic content first means nothing ever matches and every call pays full price.

> **The guide's out-of-scope list excludes "prompt caching implementation details (beyond knowing it exists)."** Know what it is and why ordering matters; don't spend study time on TTLs or breakpoint limits.

### Exam traps for 5.1

- Asking for a bigger context window or a bigger model instead of managing the budget.
- Any vector-database / semantic-retrieval answer for facts that must be present *every* turn.
- Replacing structured values with prose anywhere.
- **Instructing the model to "preserve all numerical values when summarising"** instead of moving them out of the summarised history.
- Selectively truncating history rather than using a facts block plus summarisation.
- Summarizing persistent facts along with ephemeral discussion.
- Trimming or compacting *after* the context is already full.

---

## 5.2 Escalation and ambiguity resolution

Only 4 questions in the bank, but the pattern is exam-wide — and **the official lesson frames this subtopic differently from the bank.** The bank tests *assume vs. ask* for vague requests; the guide tests **escalation calibration for support agents.** Know both.

### The three valid escalation triggers

| # | Trigger | Rule |
|---|---|---|
| **1** | **Customer explicitly requests a human** | **Escalate immediately. An absolute rule with no exceptions.** No investigation, no "let me see if I can help first" |
| **2** | **Policy exception or gap** | The request falls outside documented policy and needs human judgement |
| **3** | **Inability to make meaningful progress** | The catch-all — **but only after a genuine attempt.** "I might not be able to handle this" is not sufficient |

> **Gaps vs. violations — a distinction the exam tests.** A **violation** (a refund outside the return window) has a documented answer: *no*. A **gap** means the policy is **silent** on the situation. **Gaps require escalation; violations do not.**

### The two unreliable triggers

| Anti-pattern | Why it fails |
|---|---|
| **Sentiment-based escalation** | **Frustration does not correlate with case complexity.** A furious customer with a late delivery is easy to resolve; a calm customer asking about a policy gap needs human judgement. Sentiment measures emotional state, not difficulty |
| **Self-reported confidence scores** | **LLM self-confidence is poorly calibrated.** The model is often incorrectly confident on hard cases (it doesn't know what it doesn't know) and hedges on easy ones |

> These two are presented as plausible solutions and are **both wrong**. The proportionate first response is **explicit escalation criteria with few-shot examples in the system prompt** — prompt optimisation precedes architectural changes like classifier models.

### The frustration nuance

| Situation | Response |
|---|---|
| Straightforward issue, customer frustrated | **Acknowledge the frustration and resolve.** *"I understand this is frustrating. I can process your replacement right now."* Do **not** escalate |
| Customer reiterates wanting a human **after** you offer help | **Now escalate.** They were given the chance and declined |
| Customer says "I want a human" **from the start** | **Escalate immediately.** Zero investigation |

### Ambiguous entity matching

> A name search returns three "John Smith" records.

**Ask for additional identifiers** — email, phone, order number.

**The agent must NOT** select the most recent record, the most active record, or use **any** heuristic. Picking wrongly risks **privacy violations** (exposing one customer's data to another) and **incorrect actions** (refunding the wrong account). **The only safe response to an ambiguous match is to ask.**

### The bank's framing: assume, state, invite correction

> An assistant receives *"book a venue for the party"* with no date, guest count, budget, or venue type. Asking **4+ clarifying questions** before acting produces a **35% abandonment rate**. But when the questions are simply reduced, users get recommendations that don't match their preferences — wrong size, wrong date, wrong price range.

The scenario deliberately rules out both extremes. The answer is the middle:

1. **State explicit assumptions** drawn from the conversation — *"I'm assuming next Saturday, around 40 guests, moderate budget under $2,000."*
2. **Proceed with reversible recommendations** — show matching venues now.
3. **Invite correction** — *"Let me know if any of those assumptions are off."*
4. **Reserve clarifying questions for irreversible actions** — confirming the booking, processing payment.

**The decisive variable is reversibility.** Showing options costs nothing to undo. Charging a card does.

Why each alternative fails:

| Distractor | Problem |
|---|---|
| A structured intake form collecting all parameters upfront | Recreates the interrogation that caused 35% abandonment. Users leave before receiving any value |
| Proceed with defaults **without stating them** | The assumption is the load-bearing part. Silent wrong defaults produce mismatched results and destroy trust |
| A single compound question ("What date, guest count, and budget?") | Lowers the turn count but keeps the friction — it still demands every parameter before delivering anything |
| Context-management answers (sliding window, retrieval) | Off-topic. This is a clarification-vs-recommendation trade-off, not a context problem |

### The exception: ask when the options are materially different

> A first message says *"Set up my focus music."* This maps to three genuinely different actions the system supports: configure preferences, create a playlist, or play music immediately.

**Fix: ask one targeted clarifying question** — *"Would you like me to play focus tracks now, or set up a configuration for later?"*

This is the legitimate case for asking. Note what distinguishes it from the venue scenario:

| | Venue booking | Focus music |
|---|---|---|
| Missing information | Parameters (date, size, budget) | **Which action entirely** |
| Can you assume? | Yes — show options, adjust | No — the three actions are not variations of each other |
| Cost of a wrong guess | Low (browse different options) | Higher (wrong artifact created, or music starts unexpectedly) |
| Right move | Assume, state, invite correction | **One targeted question** |

- *Create a playlist immediately and notify the user* — performs one of three possible actions with no evidence.
- *Begin preference configuration (genres, tempo, artists)* — presumes configuration **and** adds an interrogation.
- *Play tracks immediately and let the user redirect* — pushes the recovery burden onto the user.

### The decision rule

```
Is the action irreversible, or do the options lead to
materially different actions?
      │
      ├── Yes → Ask ONE targeted question
      │
      └── No  → Assume from context
                State the assumption explicitly
                Act (reversibly)
                Invite correction
```

**Never: a pile of questions. Never: a silent guess.**

### Exam traps for 5.2

- **Attempting to resolve before honouring an explicit human request.** Absolute rule — escalate immediately.
- **Sentiment-based escalation** — frustration doesn't correlate with complexity.
- **Self-reported confidence as an escalation signal** — poorly calibrated.
- **Selecting from ambiguous matches heuristically** (most recent, most active) — privacy and correctness risk.
- Deploying a classifier model before trying explicit criteria + few-shot examples in the prompt.
- Treating a policy **violation** as a gap (violations have a documented answer).
- Always asking, or always guessing, on vague requests.
- Treating a compound question as "fewer questions."
- Assuming silently without stating the assumption.

---

## 5.3 Error propagation in multi-agent systems

**Zero questions in the bank — confirmed, not a classification artifact.** This section is written from the official lesson. Two authored questions covering it are in [5.0](../../categorized_exam_dump/domain-5-context-management-reliability/5.0-cross-topic-and-domain-level.md) (D5-G1, D5-G6).

> **The guide's framing is narrower than "validate everything."** This subtopic is specifically about **how a failing subagent reports failure to the coordinator**, and what the coordinator can do with that report. Three things are tested: structured error context, two named anti-patterns, and one distinction most developers get wrong.

### Structured error context — the four required elements

When a subagent fails it must return context that lets the coordinator make an **intelligent recovery decision**, not just know that something broke.

| Element | Contents | Why |
|---|---|---|
| **1. Failure type** | `transient` (timeout, rate limit — may succeed on retry) · `validation` (bad input — fix the query) · `business` (rule violation — escalate or find an alternative) · `permission` (access denied — cannot be retried without authorisation changes) | Determines whether retry is even meaningful |
| **2. What was attempted** | The specific query, the parameters, the target system | *"Searched academic database for 'renewable energy policy', 2022–2024"* is actionable. *"Search failed"* is not |
| **3. Partial results gathered before failure** | Whatever was retrieved before the failure | If three of five sources came back before the timeout, those three are valuable. Discarding them because the overall operation failed is waste |
| **4. Potential alternative approaches** | The subagent's own suggestions | The subagent knows its domain — try a different database, broaden the terms, use cached results |

```json
{
  "status": "partial_failure",
  "failureType": "transient",
  "attemptedAction": { "tool": "search_academic_db",
                       "query": "renewable energy policy",
                       "dateRange": "2022-2024" },
  "partialResults": [ { "title": "EU Renewable Energy Directive 2023",
                        "source": "EUR-Lex", "retrieved": true } ],
  "alternativeApproaches": [
    "Retry with narrower date range (2023-2024)",
    "Search alternative database: government_publications",
    "Use cached results from previous research session" ]
}
```

This gives the coordinator everything it needs to choose: **retry the same query, try an alternative, proceed with partial results, or escalate.**

**Trap: a generic status after retry exhaustion** — *"search unavailable"* hides the query, the partial results, and the alternatives. Generic statuses prevent informed recovery.

### The two anti-patterns

Both are tested explicitly, and they are catastrophic in different directions.

#### Silent suppression — the worst anti-pattern

Returning empty results **marked as success**: `{ "results": [], "status": "success" }` after a timeout.

The coordinator believes the search ran and found nothing. It won't retry, won't try alternatives, and produces a synthesis that silently omits an entire area. **The final output looks complete. It is missing critical content.**

This is dangerous precisely because it is *invisible* — the output looks correct and has gaps nobody can detect. In a support context it means telling the customer they have no account when the order lookup was simply down.

#### Workflow termination

Killing the entire pipeline on a single failure. One subagent times out and the whole run crashes — the other four completed successfully and their results are thrown away. A disproportionate response that wastes completed work and offers no recovery path.

**The correct middle ground is structured error propagation:** the failing subagent reports what happened, the coordinator assesses the damage, and the system continues with partial results or targeted recovery.

### Access failure vs. valid empty result

**The distinction the exam tests directly**, and the one most people get wrong.

| | Access failure | Valid empty result |
|---|---|---|
| What happened | The tool **could not reach** the source — timeout, connection error, permission denial | The tool **reached the source and ran the query**. It found no matches |
| Did the query execute? | **No** | **Yes** |
| Is this the answer? | No | **Yes — this IS the answer** |
| Retry? | Consider retry, same or modified parameters | **No.** The system worked correctly |

```python
# Access failure — consider retry
{"status": "error", "failureType": "transient",
 "message": "Connection timeout after 30s", "shouldRetry": True}

# Valid empty result — no retry needed
{"status": "success", "results": [],
 "message": "Query executed successfully. No matching records found.",
 "shouldRetry": False}
```

Conflating them causes two failures in opposite directions:

- Treating **access failures as valid empty results** → you never retry when you should (this is silent suppression).
- Treating **valid empty results as access failures** → you waste time retrying a query that will always return nothing.

### Local recovery before propagation

Subagents should handle their **own** transient failures locally — retry logic, fallback sources, degraded responses — and propagate only what they cannot resolve. When they do propagate, always include what was attempted and any partial results.

This keeps the coordinator simple: it doesn't need retry logic for every transient failure of every subagent, only a policy for persistent ones.

### Coverage annotations

When a synthesis agent combines findings, the output should note **which areas are well-supported and which have gaps**:

> *"Section on geothermal energy is limited due to unavailable journal access during research."*

Without this, a gap looks like the topic **wasn't relevant** rather than the source being unavailable. Coverage annotations tell the consumer what the report covers fully and where the known limitations are.

### Related principles that belong here

- **Guarantee at the boundary.** Domain 1's post-loop orchestration check is the same principle applied to termination rather than to data.
- **A failed step does not void the successful ones.** Domain 2's partial-progress rule is this subtopic's workflow-termination anti-pattern, seen from the tool side.
- **Surface, never silently reconcile.** Domain 4's consistency-check rule is silent suppression applied to data rather than to errors.

### Exam traps for 5.3

- **Catching a timeout and returning empty results marked successful.** The worst anti-pattern — it prevents all recovery.
- **Terminating the pipeline when one subagent fails**, discarding the others' completed work.
- Returning a generic *"unavailable"* status that hides the query, partial results, and alternatives.
- **Retrying a valid empty result** because it looks like a failure.
- Discarding partial results because the overall operation failed.
- Silently omitting a topic the research couldn't cover, instead of annotating the gap.

---

## 5.4 Codebase exploration and context degradation

Three questions, both about **spending a limited budget intelligently** rather than reading everything.

### Resolve aliases before searching for callers

> An engineer asks the agent to find all callers of a function before removing it. `calculateTax()` is defined in a core library, but it is also **re-exported through wrapper modules under domain-specific names** — the `orders` module exports it as `computeOrderTax()`.

**Fix: read the library and wrapper modules first to identify every exposed name and alias, then grep for each name across the codebase.**

The two steps are both required, in that order. You cannot grep for a name you don't know exists.

- *Grep for `calculateTax` only* — **the trap.** It returns a clean-looking, confidently incomplete list. Every `computeOrderTax` call site is missed, and removing the function breaks them at runtime.
- *Search the project documentation for usage patterns* — docs are stale and rarely cover internal usage.
- *Grep for files importing the library or wrappers, then read each one* — you read many files that import the module without using the function, which is wasteful, and you still miss qualified or dynamic invocations.

**Generalize: resolve indirection before searching.** Re-exports, aliases, wrappers, and dynamic dispatch all break naive name search.

### Start from the interface, then trace implementations

> The agent must understand how caching works before adding a new invalidation trigger. Grep shows the caching logic spans **15 files** — decorators, middleware, service classes — roughly **8,000 lines**.

**Fix: (1) analyze imports and class hierarchies to find the base cache class or interface, (2) read that core file first to understand the shared contract and design pattern, (3) trace the specific invalidation implementations in the remaining files with that structure already in mind.**

The order matters: a structural mental model first makes every subsequent file cheaper to understand.

| Distractor | Why it fails |
|---|---|
| Load all 15 files sequentially | No framework while reading; each implementation is harder to understand in isolation; burns the token budget on full reads |
| Prioritize the largest files first | **File size doesn't correlate with architectural importance.** The base class defining the interface may be 40 lines. This optimizes for volume over structure |
| Grep for `invalidate` / `expire` and read only those line ranges | You see invalidation call sites without understanding the cache lifecycle or concurrency guarantees — exactly what you need to add a new trigger safely |

**Generalize: read for structure first, then for detail. Start from the base class or interface, not from the biggest file or the most matches.**

### Delegate bounded questions

When exploration is degrading the main agent's context (see Domain 1's `VulkanPipeline` scenario), spawn subagents for specific bounded questions — *"find the test files," "trace the refund flow"* — while the main agent coordinates. Each subagent burns its own context, not yours.

### Exam traps for 5.4

- Grepping for one name when re-exports and aliases exist.
- Reading everything, or reading by file size.
- Keyword-matching without understanding the surrounding design.

---

## 5.5 Human review and confidence calibration

Six questions. The theme: **route by uncertainty, validate by segment, and keep measuring what you stopped reviewing.**

### The aggregate metrics trap — the most dangerous misconception here

> A system reports **97% overall accuracy**. The team celebrates and approves automation for all high-confidence extractions.

What the aggregate hides:

| Document type | Accuracy |
|---|---|
| Standard invoices | **99.5%** |
| Handwritten receipts | **60.1%** |
| Scanned PDFs | **72.4%** |
| International formats | **45.2%** |
| **Aggregate** | **97.0%** |

The aggregate looks excellent **because standard invoices dominate the volume**. Three document types have unacceptable accuracy, concealed by a volume-weighted average — and those segments are often where errors have the **highest** business impact (field-staff receipts, new international suppliers, scanned compliance documents).

> **The rule: validate accuracy by document type AND field segment before automating. Never automate on aggregate metrics alone.**

### Validation before automation — the sequence matters

1. **Measure accuracy by document type and field segment** — not aggregate.
2. **Calibrate confidence scores** using labelled validation sets.
3. **Set calibrated thresholds** for automation vs. human review.
4. **Implement stratified random sampling** for ongoing verification of the automated tier.
5. **Only then** reduce human review, on segments with demonstrated validated accuracy.

**Skipping to step 5 on the strength of an aggregate is the trap.** Every step exists to prevent a specific failure mode.

### Field-level confidence calibration

Raw model confidence is **relative, not absolute**. The same reported score means different things for different fields:

> *"When the model reports 0.90 on **date** fields it is actually correct **94%** of the time. When it reports 0.90 on **amount** fields it is correct **82%** of the time."*

Calibration requires **labelled validation sets** — run documents with known-correct extractions, compare reported confidence against actual accuracy, and build a calibration curve per field type per document type. Then route:

| Band | Route |
|---|---|
| Above the calibrated threshold | Automated (**with stratified sampling**) |
| Below the calibrated threshold | Human review |
| The ambiguous zone | **Prioritised** human review |

### Reviewer capacity prioritisation

Route the **highest-uncertainty** items first: low model confidence, ambiguous or contradictory sources, document types with historically poor accuracy, fields where the model expresses multiple possible interpretations.

**Do not spread reviewer capacity evenly.** An even distribution wastes time on high-confidence items the model handles well while starving the uncertain ones that actually need judgement.

**The prioritisation must be dynamic, not static.** When a reviewer finishes an item, the next should be the **highest-uncertainty item remaining** — never simply the next in chronological order.

### Route by risk signals, not by category

> An extraction pipeline validates against a JSON schema. Reviewer capacity can handle about **5%** of total volume. Which extractions should be routed to review?

**Route on direct risk signals: (1) low confidence scores, and (2) conflicting evidence in the source** — comparison tables, appendices, ambiguous phrasing, multiple plausible interpretations.

These are the strongest predictors of a wrong value. Compare:

- *Random sampling alone* — ignores available signal and spends scarce capacity on easy cases. (Note: random sampling has a *different*, legitimate job — see the next scenario.)
- *Review only when downstream systems report failures* — purely reactive; the error has already propagated.
- *Always review high-priority entity types (financial figures, dates) regardless of confidence* — the plausible trap. It over-reviews confident, unambiguous financial fields while under-reviewing genuinely ambiguous values in "lower-priority" fields. **Entity type is a proxy for importance, not for risk.**

### Keep sampling what you stopped reviewing

> The system routes extractions below **85%** confidence to human review. A quarterly audit finds that **12% of high-confidence (≥85%) extractions also contain errors** — plausible-but-wrong values. The sources vary: competitor specs pulled from comparison tables, appendices referencing different product variants, ambiguous phrasing. Nothing currently measures these residual errors.

**Fix: stratified random sampling of a fixed percentage of high-confidence extractions each week (e.g. 2–5% of the ≥85% group).**

Two things this buys you: an **ongoing measurement** of the residual error rate, and detection of **novel failure patterns** you haven't thought of yet.

- *Heuristic rules flagging documents containing tables or appendices* — only catches the failure modes you already know about. New patterns stay invisible, and that's precisely what random sampling exists to find.
- *Lower the confidence threshold to 70%* — more review workload, and still **zero measurement** of the ≥85% group. It shifts the boundary instead of measuring behind it.
- *Re-extract everything twice and flag disagreements* — **the subtle one.** It misses errors where both passes are *identically wrong* — which is exactly what happens when a comparison table reliably misleads the extractor in the same direction both times. Agreement is not accuracy.

**Principle: confidence-based routing creates a blind spot by design. Random sampling is how you keep watching it.**

### Validate per segment before automating

> After **3 months at 100% human review**, aggregate analysis shows extractions with confidence ≥90% are **97% accurate**. The plan is to automate those. What is the critical validation step before deploying?

**Analyze accuracy by document type and by field, to confirm high-confidence extractions perform consistently across all segments — not just in aggregate.**

97% overall is entirely compatible with 82% on financial figures or 71% on one document type. Automating would silently push those segments into production unreviewed.

- *Compare 85% / 90% / 95% thresholds* — useful for tuning, but secondary. Threshold tuning on aggregate data inherits the same blind spot.
- *Pilot by routing 25% of high-confidence extractions straight through* — tests in production with real consequences, and may not surface a weak segment at all if it's a small share of volume.
- *Verify the aggregate accuracy meets downstream requirements* — **aggregate metrics are exactly the thing hiding the problem.** Also, different field types often have different accuracy requirements.

**Principle: aggregate accuracy hides weak segments. Always disaggregate before automating.**

### Make the routing decision explicit in the tool output

> A document extraction tool returns ML confidence scores (0.0–1.0) per field. In production: the agent proceeds with low-confidence extractions that are wrong **23%** of the time, **and** unnecessarily requests human review for **31%** of extractions that are actually correct.

Both error directions at once is the signature of **asking the model to interpret raw numbers against an unstated threshold**.

**Fix: structure the tool output to carry the decision, not just the data:**

```json
{
  "fields": {"amount": {"value": 1299.00, "confidence": 0.62},
             "date":   {"value": "2024-05-02", "confidence": 0.71}},
  "requires_review": true,
  "review_reasons": ["low_confidence_on_amount", "ambiguous_date_format"]
}
```

Three parts, each doing a job:

1. **Raw confidence per field** — preserved for later analysis and threshold tuning.
2. **A computed `requires_review` boolean** — applies *tested* thresholds, so routing is deterministic instead of inferred.
3. **A `review_reasons` array** — tells the reviewer *which* field and *why*, so they know where to look and you can improve thresholds later.

Why the alternatives fall short:

- *Return raw scores plus few-shot examples in the system prompt* — still asks the model to interpret confidence ranges, which is the behaviour producing both the 23% and the 31%.
- *Return a single aggregate score plus a summary* — hides which field is problematic; one weak field is masked by an otherwise strong extraction.
- *Return `verified` vs. `needs_verification` objects* — closest, but without `review_reasons` the agent (and the human) doesn't know *why*, which limits both investigation and future threshold tuning.

**Principle: when a tool knows the answer, it should return the decision — not the raw inputs and a hope the model computes it the same way every time.** (Same idea as Domain 2's structured error payloads.)

### Exam traps for 5.5

- **Using aggregate accuracy to justify automating all high-confidence extractions.**
- **Only sampling low-confidence extractions** — the automated tier is where an unnoticed novel error pattern lives.
- **Using raw model confidence without calibration** against labelled sets.
- **Spreading reviewer capacity evenly** instead of prioritising by uncertainty.
- Serving the review queue in chronological rather than uncertainty order.
- Routing review by entity type or at random when uncertainty signals exist.
- Double extraction as an error check (both passes can be identically wrong).

---

## 5.6 Information provenance and multi-source synthesis

Nine questions, all about one pipeline and what gets lost as it narrows:

### The claim-source mapping — five required fields

| Field | Contents |
|---|---|
| **Claim** | The specific assertion |
| **Source URL** | Where it was found |
| **Document name** | The title of the source |
| **Relevant excerpt** | The **specific passage** supporting the claim |
| **Publication date** | When published, or when the data was collected |

> **Attribution dies during summarisation.** A synthesis agent naturally compresses and paraphrases. Without explicit instruction to preserve mappings, it produces *"Investment in renewable energy has grown significantly"* — no amount, no source, no date.

Attribution must survive **every** step: research collects mappings → analysis adds assessment **preserving** them → **synthesis merges them** → report generation emits inline citations. **Step 3 is the most common failure point.**

### Conflict handling — annotate, never resolve

> Source A reports 12% market growth. Source B reports 8%. Both are credible publications.

**Correct: annotate both values with full source attribution and let the consumer decide.**

```markdown
Market growth estimates vary by source:
- **12% growth** — IEA World Energy Report (published June 2024, 2023 calendar year data)
- **8% growth**  — Bloomberg NEF Annual Review (published March 2024, Jul 2022–Jun 2023 data)

The difference may reflect different reporting periods and methodologies.
```

| Wrong approach | Why |
|---|---|
| **Select the more recent source** | Arbitrary; destroys information and presents false certainty |
| **Average the values** | Invents a figure no source supports |
| **Pick the more authoritative publisher** | Still arbitrary selection |
| **Escalate to a human before reporting either** | Over-escalation — the annotated presentation *is* the answer |

### Analysis completes with conflicts intact

When document analysis hits conflicting values it must **finish its work with the conflict included and annotated** — it does **not** resolve it. That decision belongs to the coordinator or the consumer.

```json
{ "field": "annualRevenue", "conflictDetected": true,
  "values": [
    { "value": "$4.2M", "source": "Annual Report 2023",
      "context": "Audited statements, FY ending December 2023" },
    { "value": "$3.8M", "source": "SEC Filing Q4 2023",
      "context": "Preliminary unaudited, calendar year 2023" }],
  "possibleExplanation": "May reflect audited vs preliminary figures and fiscal vs calendar year" }
```

### Temporal awareness turns contradictions into trends

> Source A (2023): 8% growth. Source B (2024): 12% growth.

**Without dates these look contradictory. With dates they tell a story** — growth accelerated from 8% to 12%. The "conflict" is a **trend**.

**Require publication / data-collection dates in all structured outputs.** Without temporal context, valid trends get misread as data-quality problems, and the synthesis agent may flag or suppress findings that are actually consistent.

### Content-appropriate rendering

| Content type | Format | Why |
|---|---|---|
| **Financial data** | **Tables** | Numbers, comparisons, and trends are most readable tabulated |
| **News and current events** | **Prose** | Narrative context and cause-and-effect read naturally as paragraphs |
| **Technical findings** | **Structured lists** | Architectural patterns, API specs, config options are clearest bulleted with hierarchy |

**Forcing everything into one format — all tables, all prose, or all lists — degrades readability.**

### Established vs. contested

Reports should include **explicit sections distinguishing well-established findings from contested ones**, preserving each source's original characterisation and methodological context. **A finding supported by three independent sources is different from one based on a single report**, even when both read with equal confidence.

```
web search (120K raw) ─► analysis (15K insights) ─► synthesis (3K narrative) ─► report
                            ▲                           ▲
            metadata is dropped here ───────── and here
```

**Every compression step is where provenance dies unless you carry it separately.**

### Structured claim–source mappings

> In a multi-agent research system, final reports frequently contain claims with no attribution. Investigation shows the subagents **do** attach citations correctly — but the **synthesis agent loses track of which sources support which conclusions** when it combines findings.

**Fix: require every subagent to emit structured claim–source mappings, and require the synthesis agent to preserve and merge them.**

```json
{"findings": [
  {"claim": "Adoption reached 35% in Q3 2024",
   "source_url": "https://...",
   "source_excerpt": "...35 percent of installed capacity...",
   "date": "2024-Q3"}
]}
```

The failure is at the *merge* step, so the fix has to make the mapping a first-class structure that survives merging.

- *Semantic similarity matching to reconstruct citations afterwards* — approximate and lossy. A synthesized sentence can match several passages; claims get attributed to the wrong source, confidently.
- *Inline source prefixes in the text* (`"SOURCE_1: ..."`) parsed at report time — fragile. Synthesis rewrites text, and rewriting strips or misaligns prefixes.
- *A citation-resolution agent that analyzes the interaction transcripts* — infers attribution indirectly from logs instead of just carrying it directly. Complexity substituting for structure.

**Principle: attribution must be carried forward as structure, never reconstructed after the fact.**

### Separate content from metadata

> The pipeline compresses 120K tokens of raw content → 15K insights → a 3K narrative. Reports make factual claims **without citations** because source metadata — URLs, document names, page numbers — is lost during the summarization steps.

**Fix: have each agent output two separate portions — (1) the content summary, and (2) a metadata portion carrying source URLs, document IDs, page numbers, and data collection dates. The synthesis agent passes both through; the report generator receives the synthesis draft plus a structured source index.**

Separation is what makes this work: summarization compresses the *content* channel and leaves the *metadata* channel untouched.

- *Skip summarization and pass full raw outputs to the report generator* — preserves provenance by reintroducing the 120K-token bloat the pipeline exists to avoid.
- *Instruct the synthesis agent to embed references inline in the prose* ("according to source X") — informal and inconsistent; loses exact URLs and page numbers.
- *Query the web search agent retrospectively to re-locate sources* — latency, and the report generator has to guess which claims need re-locating. It also misses findings that came from document analysis rather than search.

### Passing context to the report generator

Same pipeline, framed as a context-passing question. **Pass the 3K synthesis draft plus a structured source index mapping key claims to URLs and supporting excerpts.**

| Option | Verdict |
|---|---|
| Draft + structured source index | ✅ Exactly what's needed for accurate citation, nothing more |
| Draft only, match claims to sources afterwards | ❌ Post-hoc semantic matching misattributes |
| Full accumulated context from every stage | ❌ Complete, and reintroduces the bloat |
| A condensed summary naming sources by name only | ❌ No URLs or page numbers; readers can't verify |

### Dates turn contradictions into trends

> The synthesis agent researching renewable energy adoption receives **35% (2024)** from web search and **18% (2022)** from an internal report. It flags them as **contradictory** instead of recognizing growth over time.

**Fix: require subagents to include publication or data-collection dates in their structured outputs**, so the synthesis agent can recognize a temporal sequence and explain the trend.

```json
{"finding": "35% adoption", "date": "2024-Q3", "source_url": "..."}
{"finding": "18% adoption", "date": "2022-Q1", "source_url": "..."}
```

- *Treat the most recent data as authoritative and put older findings in an appendix* — silently discards the older observation and never explains *why* the values differ. The trend — the actual insight — is lost.
- *Automatically discard old data when newer data exists* — **false certainty.** The 2022 survey may use different inclusion criteria than the 2024 one, in which case neither supersedes the other and the difference isn't a trend at all. You can't know without the metadata.
- *Restrict web search to the last 6 months* — arbitrarily excludes relevant history and doesn't address the root cause.

**Principle: without dates, a time series is indistinguishable from a contradiction.**

### Preserve how each source characterized its own uncertainty

> Two conflicting findings: web search returns *"Industry analysts estimate a $50B market size (methodology varies)"*; document analysis returns *"Peer-reviewed study estimates $35B (±$7B, 95% CI)."* The synthesis agent either picks one arbitrarily (losing nuance) or over-hedges into uselessness ("the market may be $35B–$50B depending on various factors").

**Fix: structure the report with explicit sections distinguishing well-established findings from contested ones, preserve each source's original characterization and methodological context, and explain *why* the estimates differ** — different methodologies, different populations, different timing.

The two estimates are not commensurable. One is a formal confidence interval from a peer-reviewed study; the other is an informal industry figure with acknowledged methodological variation. There is no correct way to combine them into a single number, so the report must present them as what they are.

- *Normalize uncertainty to 0.0–1.0 scores and compute a confidence-weighted average* — **the most seductive distractor.** It produces a precise-looking number from incomparable inputs. A 95% CI and a "methodology varies" caveat cannot be placed on one scale, and the output's precision is entirely false.
- *Filter out low-confidence findings before synthesis* — removes the disagreement, producing an artificially unanimous report that hides an important caveat.
- *Add a verification subagent requiring two-source corroboration* — discards unique, valuable findings that only one source reports, and eliminates the disagreement you should be explaining.

**Principle: report well-established and contested findings in separate sections. Never average incomparable estimates.**

### Render each content type in its native format

> The research system expands to specialized sources: a financial API returning structured JSON (revenue, margins, growth rates — inherently tabular), a news monitoring agent returning prose summaries, and a patent analysis agent returning structured lists of technology areas. The synthesis agent converts everything to bullet points. Financial comparisons lose tabular clarity; news summaries lose narrative flow.

**Fix: render each content type appropriately** — financial data as tables/matrices, news as flowing prose, technology areas as structured lists or trees.

Format carries semantics. A table expresses that the same metrics are compared across entities; bullets destroy that relationship. Prose expresses causal and temporal narrative; bullets fragment it.

- *Standardize all outputs to JSON with uniform fields* — improves programmatic consistency, does nothing for presentation clarity. Flattened financial data in uniform JSON still loses the tabular relationship.
- *Add a format-conversion layer to a common intermediate like Markdown* — sounds flexible, but it's the same flattening with an extra step.
- *Standardize everything to prose* — loses tabular precision **and** the scannability of structured lists. The same mistake in the opposite direction.

**Principle: preserve each source's native representation; don't flatten to a uniform intermediate.**

### Exam traps for 5.6

- Reconstructing citations after synthesis instead of carrying mappings through.
- Discarding older data instead of dating it.
- Averaging or weight-averaging incomparable estimates into false precision.
- Filtering out disagreement to make a report look unanimous.
- Flattening every source format into one uniform representation.

---

## The six sentences that answer most of Domain 5

1. **Prose is lossy for data.** Exact values live in a structured block, not a summary.
2. **Preserve facts, compress discussion.** Progressive summarization for old turns, verbatim for recent ones, and a persistent facts block that summarization never touches.
3. **Trim tool output before it accumulates**, and ask upstream agents for structured findings rather than raw content.
4. **Ask only when the action is irreversible or the options differ materially.** Otherwise: assume, state the assumption, act, invite correction.
5. **Route review by uncertainty, validate by segment, and keep sampling what you stopped reviewing.**
6. **Carry provenance — sources and dates — through every compression step.** Without dates a trend looks like a contradiction; without sources a claim has no citation.

## Study order within this domain

1. **5.1 context window management** — 14 questions; learn the hybrid strategy and why vector retrieval is almost always the wrong answer.
2. **5.6 provenance** — 9 questions; claim–source mappings, dates, and never averaging incomparable estimates.
3. **5.5 confidence calibration** — 6 questions; risk-based routing, segment validation, continuous sampling.
4. **5.2 ambiguity** — 4 questions; the reversibility test.
5. **5.4 codebase exploration** — 3 questions; resolve aliases, read for structure first.
6. **5.3 error propagation** — **0 questions in the bank but a live blueprint subtopic.** Read the lesson, then do D5-G1 and D5-G6 in [5.0](../../categorized_exam_dump/domain-5-context-management-reliability/5.0-cross-topic-and-domain-level.md). This is the largest single blind spot in the whole bank.
7. **[5.0 cross-topic questions](../../categorized_exam_dump/domain-5-context-management-reliability/5.0-cross-topic-and-domain-level.md)** — eight authored scenarios, two of which fill the 5.3 gap.

## Links

[5.1 Context window management](https://claudecertificationguide.com/learn/5-context-management/5-1-context-window-management) ·
[5.2 Escalation & ambiguity](https://claudecertificationguide.com/learn/5-context-management/5-2-escalation-ambiguity) ·
[5.3 Error propagation](https://claudecertificationguide.com/learn/5-context-management/5-3-error-propagation) ·
[5.4 Codebase exploration](https://claudecertificationguide.com/learn/5-context-management/5-4-codebase-exploration) ·
[5.5 Human review & calibration](https://claudecertificationguide.com/learn/5-context-management/5-5-human-review-calibration) ·
[5.6 Information provenance](https://claudecertificationguide.com/learn/5-context-management/5-6-information-provenance)
