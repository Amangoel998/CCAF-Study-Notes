# Domain 2 — Tool Design & MCP Integration

**Blueprint weight: 18% (~11 of 60 questions) · 53 questions in the bank, ~30 distinct concepts**

> Heavy duplication: the provenance-in-the-description scenario alone appears four times, and the structured-error-metadata scenario four more. See [2.0 Cross-Topic & Domain-Level](../../categorized_exam_dump/domain-2-tool-design-mcp-integration/2.0-cross-topic-and-domain-level.md) for the clusters and the placement audit.

> **The one sentence that governs this domain:** the schema, the description, and the error payload *are* the instructions. If the model picks the wrong tool, invents a parameter, or retries something that will never succeed, you fix the **interface** — not the prompt.

Almost every question here follows the same shape: a tool misbehaves, and four options are offered. One changes the tool's *structure*. One adds examples or instructions to the *prompt*. One adds *validation after the fact*. One builds *new infrastructure*. The structural fix wins nearly every time, because the model only ever sees the schema and description when it decides what to call — it does not see your validation code, and prompt text does not travel with the tool to other applications.

A useful ranking to carry into every question in this domain:

| Rank | Fix type | Why it wins or loses |
|---|---|---|
| 1 | **Make the bad call structurally impossible** (split the tool, require a token, hide the tool until discovered) | The model cannot emit what the schema cannot express |
| 2 | **Put the information in the schema / description** (provenance, enums, policy, output contract) | Visible at decision time; travels with the tool |
| 3 | **Return better structured results and errors** | Lets the model self-correct on the next turn |
| 4 | Few-shot examples in the system prompt | Advisory; lives outside the tool; doesn't travel |
| 5 | Server-side validation only | Catches the bad call *after* it's made; the waste still happens |

---

## 2.1 Tool interface design

The densest subtopic in the domain (20 questions). It divides into four recurring problems: **ambiguous inputs**, **unusable outputs**, **overloaded tools**, and **unsafe or racy operations**.

> Two questions were re-filed into this subtopic: one about enforcing a monetary approval threshold (previously 1.4, because it reads as a workflow rule) and one about parameter descriptions (previously 2.2, because the symptom is a failed call). In both, the correct answer changes the **interface**, which is what places them here. Expect the exam to disguise interface questions as workflow or error-handling questions.

### Descriptions are the PRIMARY selection mechanism

Not supplementary metadata, not an afterthought. The model reads descriptions to decide what to call.

**A production-grade description has five elements:**

| # | Element |
|---|---|
| 1 | **What the tool does** — its purpose, stated unambiguously |
| 2 | **What inputs it expects** — types, formats, constraints, required vs. optional |
| 3 | **Example queries it handles well** — concrete anchors |
| 4 | **Edge cases and limitations** — what it does **not** do |
| 5 | **Explicit boundaries** — when to use **this** tool versus similar ones |

```text
❌ Minimal (causes misrouting)
get_customer: "Retrieves customer information"
lookup_order:  "Retrieves order details"

✅ Production-grade
get_customer: "Looks up a customer account by email, phone, or customer ID.
  Returns profile (name, contact, account status, loyalty tier). Use when you
  need to verify who the customer is. Do NOT use for order-specific queries —
  use lookup_order for those."

lookup_order: "Retrieves order details by order number (format #NNNNN) or
  tracking ID. Returns status, items, shipping, refund eligibility. Use when a
  customer asks about a specific order. Do NOT use for identity verification —
  use get_customer for that."
```

Element 5 does the heavy lifting. **"Do NOT use for X — use Y for that"** is the construction that resolves misrouting.

### The misrouting fix hierarchy

> This is the exam guide's **sample Question 2**. Four plausible options, three wrong.

| Fix | Verdict |
|---|---|
| **Expand the descriptions** | ✅ **Correct.** Low effort, high leverage, addresses the root cause |
| Few-shot examples in the system prompt | ❌ Token overhead without fixing *why* the model is confused — treats the symptom |
| A routing classifier that pre-selects the tool | ❌ Over-engineered as a first step; bypasses the model's language understanding and adds infrastructure |
| Consolidate the two tools into one | ❌ Valid long-term architecture, but far more effort than rewriting two descriptions |

> **The generalizable principle: the exam consistently favours low-effort, high-leverage fixes.** Better descriptions before routing classifiers. Scoped access before full access. Community servers before custom builds.

**But read the condition.** Descriptions are the fix when the agent has a *workable* number of tools and simply can't tell two apart. They are **not** the fix when the toolkit itself is the problem — past roughly 4–5 tools per agent, selection degrades on decision complexity alone, and rewriting 22 descriptions leaves that untouched. **Count the tools before choosing the remedy** (see 2.3).

### Renaming, and a subtle override

**Renaming** fixes overlap at the interface level without touching implementation — `analyze_content` → `extract_web_results` with a web-specific description.

> ⚠️ **System prompt interactions.** Keyword-sensitive instructions can **silently override well-written descriptions**. A system prompt saying *"always check customer details before proceeding"* may route any customer-adjacent query to `get_customer` regardless of what the descriptions say. **After updating descriptions, re-read the system prompt for conflicts.**

### Problem 1: the model fabricates a parameter

> The agent calls `process_refund(order_id=...)` immediately, passing a plausible-looking but **completely invented** `order_id`, instead of calling `lookup_order` first. The refund fails because the ID doesn't exist.

**Fix: state the provenance in the tool description** — "`order_id` must be obtained from a prior `lookup_order` call; never assume or construct this value."

Why fabrication happens at all: the model sees a required string parameter with no stated origin, and generating a plausible string is exactly what a language model does. The schema never told it the value had to come from somewhere.

Why the alternatives lose:

- **Server-side validation that rejects unknown IDs.** This catches the bad call *after* it has been made. The model still fabricates, still burns a turn, still has to recover. You've added a guard rail without removing the cause.
- **Pre-parsing the user's message to extract an order ID.** Fragile, and the user often never stated one.
- **Changing `tool_choice`.** Forcing a tool doesn't tell the model where a parameter's value comes from.

**Generalize:** every parameter whose value must come from somewhere specific should say so in the description. Prerequisites, provenance, dangerous side effects, and valid combinations all belong in the description — it is part of the interface, not documentation.

### Problem 2: the parameters are inherently ambiguous

> `update_game_score(game_date, home_team, away_team)` fails three ways: the agent uses team nicknames instead of official names, formats dates inconsistently, and picks the wrong game when two teams played a rematch.

**Fix: add a dedicated lookup tool that returns an unambiguous `game_id`, and have the action tool take `game_id`.**

This is the highest-value pattern in the subtopic, because it kills all three failure modes at once — the agent no longer constructs *any* of those fields.

The attractive wrong answer is **enums plus regex patterns**: constrain `home_team` to an enum of official names, constrain `game_date` to `^\d{4}-\d{2}-\d{2}$`. That genuinely fixes failures 1 and 2 — and leaves failure 3 completely untouched. Two valid, perfectly-formatted games can exist for the same matchup. **Validation can enforce format; only an identifier can enforce identity.**

**Heuristic:** when a scenario lists multiple failure modes, check whether an option fixes all of them. The exam loves offering a fix that handles two out of three.

### Problem 3: the tool returns prose instead of data

> A document search tool returns: `"Found 3 documents: Q2 Budget Proposal, Q2 Budget Forecast, Annual Review"`. The agent then needs to perform multi-step operations on specific documents.

**Fix: return structured data with a stable ID and metadata per result.**

```json
{"results": [
  {"id": "doc_8f21", "title": "Q2 Budget Proposal", "modified": "2024-04-02", "owner": "finance"},
  {"id": "doc_93ba", "title": "Q2 Budget Forecast",  "modified": "2024-04-11", "owner": "finance"}
]}
```

Titles are not handles. "Q2 Budget Proposal" and "Q2 Budget Forecast" are confusable, titles can change, and nothing downstream can address a document by its title reliably. An ID is a stable, machine-usable handle.

- *Return human-readable URLs instead* — helps a person, not the next tool call.
- *Keep prose and tell the model to parse carefully* — you've made correctness depend on string parsing.

The same idea appears in a portfolio-value tool: return explicit JSON fields rather than a formatted text string, so the agent can extract a specific value without parsing free text.

**Rule: a bare `"success"` with no ID and no detail is a broken response.** The next turn has nothing to reference.

### Problem 4: one tool doing several jobs

Two near-identical scenarios establish the split-by-operation rule.

> **Order management.** One unified tool handles refund (needs amount + reason), cancel (needs reason), and reschedule (needs shipping address). All share `order_id`. With everything optional, the agent omits required parameters and includes irrelevant ones.

**Fix: three separate tools, each requiring exactly the parameters its operation needs.**

> **Workout logging.** `log_workout(exercise_type, value, measurement)` receives nonsense combinations — reps for running, miles for bench press. Cardio uses time/distance; strength uses reps/sets. **23% of calls are invalid.**

**Fix: `log_cardio_workout(duration_minutes | distance_miles)` and `log_strength_workout(reps, sets)`.** Invalid combinations become *structurally impossible* — the model cannot express "bench press, 5 miles" because no schema accepts it.

In both, the losing answer is "keep one tool, make everything optional, add few-shot examples showing valid combinations." Examples describe which combinations are valid; a split schema makes invalid ones unrepresentable. **Prefer the fix that removes the possibility over the fix that discourages it.**

> **Generic free-text tool.** A document agent has one `analyze_document(document, free_text_instruction)` tool. "Extract key financial metrics" returns narrative summaries; "summarize the methodology" returns raw tables. **35% of results need re-requests.**

**Fix: split into purpose-specific tools — `extract_data_points`, `summarize_content`, `verify_claim_against_source` — each with a defined input and output contract.** The problem isn't that the instruction is unclear; it's that a free-text instruction field means the *output contract is undefined*. Nothing says what shape the result should take, so it varies run to run.

**When to split vs. merge:**

| Symptom | Action |
|---|---|
| Irrelevant/omitted parameters, invalid combinations, undefined output shape | **Split** by operation |
| Two tools whose descriptions genuinely overlap and the model picks wrongly | **Merge**, or sharply differentiate the descriptions |

### Problem 5: the model picks the wrong one of two similar tools

> `archive_file` and `delete_file` both exist with minimal descriptions. Policy says backup files must be archived, never deleted. When users say "remove old backups," the agent calls `delete_file`.

**Fix: expand the tool descriptions to carry the policy** — e.g. on `delete_file`: *"Permanently removes a file. Do not use for backup files; use `archive_file` instead."*

The near-miss answer is few-shot examples in the system prompt. It can work, but tool descriptions are superior for two reasons the exam cares about:

1. The description is read at the exact moment the model is choosing between the two tools.
2. The description **travels with the tool**. If the tool is exposed through another application, another agent, or an MCP server, the system prompt doesn't come with it; the description does.

### Problem 6: too many tools to choose from

> An integration platform exposes **50+ specialized API connectors**. Tool selection accuracy drops to **58%**. A `search_connectors` tool exists, but agents skip it and call connectors directly — often the wrong one.

**Fix: make connectors unavailable by default and only add them to the agent's tool list once `search_connectors` surfaces them (deferred / discovery-based loading).**

This structurally enforces search-then-use: **the model cannot call a tool it hasn't discovered.** Improving descriptions is a soft guardrail — all 50 connectors remain directly callable, so skipping search stays possible.

### Problem 7: the operation is unsafe or racy

> **Bypassed dry run.** `remove_team_member(dry_run: boolean)` is meant to be previewed first, but in **15%** of calls the agent passes `dry_run=false` directly. Policy requires explicit user confirmation before every removal.

**Fix: split into `preview_remove_member()` which returns the impact details plus a single-use confirmation token, and `execute_remove_member(token)` which requires that exact token.**

A boolean is just another parameter the model can set. A token the execute tool *cannot proceed without* binds execution to a specific previewed action.

The subtle distractor: *server-side check that an identical call was previewed within the last 60 seconds*. That sounds equivalent, but it matches on **parameters**, not on **the specific approved preview**. Two overlapping requests, a parameter the preview didn't cover, or a 61-second gap all break it. A single-use token is an exact binding.

> **Race condition.** The agent calls `get_available_slots(date, provider_id)`, then `book_appointment(provider_id, slot_time, patient_id)`. **15% of bookings fail** with "slot no longer available" because another user claimed the slot in between.

**Fix: combine into a single atomic `find_and_book_appointment()` that checks and books in one operation.** The failure window exists *because* there are two calls. Returning alternative slots on failure just reopens the same window on the retry; prompt-level retry logic does the same.

**Pattern: check-then-act across two tool calls is a race. Make it one call.**

### Problem 8: the result set is too big

> A search tool queries an external catalog with 50 items per page. Queries match 200+ products. Auto-fetching all pages causes **15–20 second delays**.

**Fix: return the first page immediately, plus the total match count and a cursor for fetching more.** The caller decides whether it needs more.

The tempting answer is to silently cap at two pages. That fixes latency and **hides truncation** — nothing signals that more matches exist, so the agent reasons as if it has seen everything. **A result set that is silently incomplete is worse than a slow one.**

### Problem 9: the schema itself is eating the context

> A tool schema has 12 fields and consumes ~2,500 tokens. Document processing shows: under 150K tokens → **98% accuracy**; at 175–190K tokens → **71% accuracy**, with misses consistently at the **end of the document**. The context limit is 200K.

**Cause: the tool schema, system prompt, and document content all draw on the same token budget.** As combined input approaches the ceiling, content near the end gets crowded out and receives insufficient attention.

The attractive distractor is "the model has an inherent attention-span limit." Three details in the scenario refute it: accuracy is fine below 150K, degrades only as total input nears the stated 200K limit, and the misses cluster at the *end* of the input rather than randomly. That's a budget problem with a positional signature, not a fixed cognitive ceiling.

**Takeaway: tool definitions are not free.** A verbose schema plus a long document competes for the same space. See 4.3 for the companion question on trimming field descriptions without losing accuracy.

> **Filing note:** the bank now files this under **5.1 Context Window Management**, because the concept is the shared token budget rather than schema design. It stays here too — if you only meet it under "context management" you will not recognize it when the stem is about a tool.

### Exam traps for 2.1

- Choosing validation (which catches the error) when the question asks you to prevent it.
- Choosing an answer that fixes two of three stated failure modes.
- **Choosing few-shot examples, a routing classifier, or consolidation to fix misrouting from weak descriptions** — expand the descriptions first.
- **Ignoring the system prompt after improving descriptions** — keyword-sensitive wording silently overrides them.
- Applying the description fix when the real problem is **too many tools** (that's 2.3).
- Accepting a boolean flag or an optional parameter as a safety control.
- Accepting a silent cap or truncation as a latency fix.

---

## 2.2 Structured error responses

16 questions, and they all orbit one idea: **the error payload is how the model decides what to do next.** An undifferentiated error forces the agent to guess, and it guesses differently every time.

### The two error layers

| Layer | Shape | Who handles it |
|---|---|---|
| **Protocol error** | A JSON-RPC `error` with a `code` and `message`. **No result object at all** | The **host application**. The model never sees a tool result, because no tool ran |
| **Tool execution error** | A normal `result` with **`isError: true`** and detail in `content` | **The model** — it reads this and decides what to do next |

A protocol error means the call was **built wrong** — unknown tool name, arguments failing the input schema, server fault. The fix belongs in the tool schema or the client, **not in the agent's recovery loop**. Everything below concerns the second layer.

### The four error categories

| Category | Cause | Recovery | `isRetryable` |
|---|---|---|---|
| **Transient** | Timeout, service unavailable, rate limit. Request valid, system briefly not | Retry after a delay | **`true`** |
| **Validation** | Invalid format, missing field, out-of-range value. The request is malformed | **Fix the input**, send a corrected call | `false` |
| **Business** | Policy violation, limit exceeded. Technically valid but violates a constraint | **Alternative workflow or escalation** | `false` |
| **Permission** | Access denied, insufficient credentials | Escalate, or retry as a principal with the right access | `false` |

```json
{
  "isError": true,
  "content": [{ "type": "text", "text": "Refund exceeds policy limit" }],
  "structuredContent": {
    "errorCategory": "business",
    "isRetryable": false,
    "description": "Refund amount of £750 exceeds the £500 automatic limit. This
      requires manager approval. Please escalate to a human agent with the details."
  }
}
```

> **Shape note:** `errorCategory`, `isRetryable`, and `description` are an **application-level convention**, not part of the MCP envelope — `CallToolResult` defines only `content`, `structuredContent`, and `isError`. They belong in `structuredContent`. **The exam guide names these four categories and the boolean, so know them by name**; just don't expect to find them in the MCP specification.

### What `isRetryable` actually signals

It answers **one narrow question: will resending this *exact* request work?** Only transient errors get `true` — everything else needs something to change first: the input (validation), the request itself (business), or the caller (permission).

**Read `isRetryable` to decide whether to resend as-is; read `errorCategory` to decide what to do when you can't.**

> ⚠️ **Two traps live in the three `false` rows.**
>
> 1. **`false` does not mean "abandon the task."** Validation is `false` because the input must change — but the agent recovers **unaided** by correcting it. Only business and permission are genuine dead ends needing an alternative path or a different principal.
> 2. **Don't mark validation `true`** on the reasoning that the agent can recover from it. It can — but not by resending the same call. `order-abc` fails the same format check every time.

### Access failure vs. valid empty result — "the distinction to nail"

> A tool returns an empty array after a customer lookup. The agent **retries 3 times, then escalates to a human.** Analysis shows the customer's account simply does not exist.

The tool **succeeded**. It queried the database, found no match, and correctly returned nothing. But because the response doesn't distinguish *"I couldn't reach the database"* from *"I reached it and found nothing,"* the agent treats both as a retriable failure.

| | Access failure | Valid empty result |
|---|---|---|
| What happened | The tool **couldn't reach** the source — timeout, auth failure, service down. The data might exist; the tool couldn't check | The tool **queried successfully** and found no matches |
| Shape | `isError: true` + `errorCategory` + `isRetryable: true` | **`isError: false`** + `resultCount: 0` |
| Retry? | Consider it | **No.** "No results found" *is* the answer |

**The fix: structure responses so a successful query with no results looks nothing like a failed query.**

```json
// Valid empty result — NOT an error
{ "isError": false,
  "content": [{ "type": "text",
    "text": "No customer found matching 'john@example.com'. The query executed
             successfully but returned no matches." }],
  "structuredContent": { "resultCount": 0 } }
```

*(The same distinction is tested from the multi-agent side in [5.3](../Domain-5-Context-Management-Reliability/summary.md#53-error-propagation-in-multi-agent-systems).)*

### The signature failure

> Every tool error comes back as `{"isError": true, "content": [{"type": "text", "text": "Operation failed"}]}`. In production the agent sometimes retries 5+ times (wasteful on a validation error), sometimes escalates immediately (premature on a transient network blip), and sometimes asks the user for clarification (inappropriate for a permission error).

**The erratic, inconsistent behaviour is the tell.** The model isn't being unreliable — it's being asked to infer a recovery strategy from a string that contains no information.

**Fix: return structured error metadata.**

```json
{
  "isError": true,
  "errorCategory": "transient",       // transient | validation | permission | not_found
  "isRetryable": true,
  "message": "Database connection timed out after 30s.",
  "suggestedAction": "Retry in a few seconds; this usually succeeds."
}
```

With `errorCategory` and `isRetryable` present, the three recovery paths become mechanical: retry, escalate, or ask.

Why the alternatives lose:

- **A separate `analyze_error` tool the agent calls after a failure.** Correct classification, delivered one round trip too late. The classification is already known at the moment of failure — deliver it inline.
- **A single `retryable` boolean and nothing else.** A real improvement, but the agent still can't distinguish a permission denial from a not-found, which need different user-facing responses.
- **Few-shot examples teaching the agent to parse error text.** You're training it to reverse-engineer a category that the tool could simply state.

### Transient vs. permanent, and who retries

> A refund tool produces two error families: technical (503s, timeouts) at roughly **5%** of calls, and business ("exceeds return window", "already refunded") at roughly **12%**. The agent wastes 3–4 turns retrying the business errors.

**Fix: retry transient errors *inside the tool* with bounded exponential backoff; surface permanent errors to Claude with a descriptive message so it can take corrective action.**

Division of labour:

| Error type | Who handles it | Why |
|---|---|---|
| Transient (503, timeout, rate limit) | **The tool**, with bounded backoff | Purely mechanical. The model adds no value and burns turns. |
| Permanent / business (already refunded, outside window, no permission) | **Surface to the model** | Needs a decision: explain to the customer, try a different path, or escalate. |

- *Retry everything* — burns turns on failures that will never succeed.
- *Hide everything behind tool-level retries* — the companion trap. If the tool silently retries a permanent error too, Claude never learns it failed or why, and can't choose an alternative recovery path.
- *Add a prerequisite eligibility-check tool* — an extra call before every refund, and the permanent error can still occur.

### Business errors need customer-usable reasons

Same refund scenario, framed around the conversation: both error families return plain text, so the agent wastes retries *and* can't give the customer a usable explanation.

**Fix: structured error responses carrying `retriable: false` plus a customer-friendly explanation** — e.g. `"This order was refunded on 2024-03-14; a second refund isn't possible."` Two jobs in one payload: stop the futile retry, and hand the agent language it can relay directly.

### Instructive messages: say what to try next

Anthropic's documented guidance, tested directly: *instead of a generic "failed", include what went wrong and what Claude should try next.*

> `lookup_order` returns `is_error: true` with the message `"Tool execution failed."` The agent either retries the identical call until the turn limit, or immediately calls `escalate_to_human` without trying alternatives.

**Fix: error-type-specific messages that name the next step.**

| Condition | Message |
|---|---|
| Order not found | `"Order not found — try get_customer to search by phone number."` |
| Database timeout | `"Database timeout (transient) — a retry should succeed."` |

Note both keep `is_error: true`. Two distractors attack opposite ends:

- *Hide every failure behind tool-level retries* — prevents Claude from choosing an alternative recovery path when retrying is the wrong move.
- *Drop `is_error: true` and return the failure as normal content* — destroys the error signal entirely; the agent may treat a failure as a successful result.

**Keep the error flag. Make the message actionable.**

### Protocol errors vs. tool result errors

The most frequently missed distinction in this subtopic.

> A check-availability tool hits three conditions: (1) the request is **malformed** — the required `user_email` is missing; (2) the external calendar API returns **404**, the user doesn't exist; (3) the external API returns **503**, temporarily unavailable.

**Correct classification:**

| Condition | Classification | Reason |
|---|---|---|
| Missing required `user_email` | **JSON-RPC protocol error** | The *call itself* is invalid. It never should have been made and cannot be executed. |
| 404 from the calendar API | **Normal tool result with `isError: true`** | The request was well-formed and executed. The *outcome* was a failure. |
| 503 from the calendar API | **Normal tool result with `isError: true`** | Same — well-formed call, runtime failure downstream. |

The boundary: **protocol errors are about the call being invalid; tool result errors are about a valid call producing a failure outcome.**

The attractive trap is classifying the 404 as a protocol error because it sounds like a "hard" failure. But the request was perfectly well-formed — the failure came from an external system, which is precisely what tool result errors exist for. And the practical consequence matters: a protocol error isn't something Claude can reason about or recover from, whereas "that user doesn't exist" is exactly the kind of outcome Claude should see and respond to.

### Partial progress: communicate, don't pretend

> A billing dispute agent successfully calls `get_customer` and `lookup_order`, then `process_refund` times out. The agent has enough information to explain the charges and confirm eligibility, but cannot complete the refund.

**Fix: transparently communicate the failure *and* preserve the partial resolution** — explain the charges, confirm eligibility was verified, acknowledge the system issue, and offer escalation or a later retry.

- *Retry with backoff until it succeeds* — leaves the conversation hanging with no explanation to the customer.
- *Escalate immediately* — throws away the useful work already done; the human restarts from zero.
- *Report only the failure* — same waste.

**Principle: a failed step does not invalidate the steps that succeeded. Report what you learned, name what failed, offer a path forward.**

### Exam traps for 2.2

- Treating every failure as one generic error — without `errorCategory`, `isRetryable`, and a description the agent cannot choose a recovery.
- **Retrying a valid empty result from a successful query.** The canonical anti-pattern.
- **Treating business errors as retryable** — the same policy violation applies every time.
- **Marking validation `isRetryable: true`** because the agent can recover from it — it can, but not by resending the same call.
- **Reading `isRetryable: false` as "abandon the task."** Validation is self-recoverable; only business and permission are dead ends.
- Classifying a downstream 404/503 as a protocol error — the call was well-formed.
- Removing the error flag so the model "reads it as text."
- Silently suppressing subagent errors by returning empty results as success.
- Discarding partial progress when a later step fails.

---

## 2.3 Tool distribution and tool choice

### The tool overload problem

**The optimal range is 4–5 tools per agent, scoped to its role.** Eighteen tools degrades selection reliability; every additional tool adds decision complexity.

**Relevance matters as much as quantity.** A synthesis agent should **not** have web search tools — given them, it will run its own searches instead of using the results already handed to it, duplicating work and burning context.

### Four remedies — diagnose before you prescribe

| Situation | Remedy |
|---|---|
| Few enough to handle, but **two of them read alike** | **Sharpen the descriptions** (2.1) |
| **Different jobs** (query, transform, export) | **Split by role**, 4–5 each |
| **Variations on one job, sharing a shape** | **Consolidate** into one parameterised tool |
| A tool can do **more than the agent should be able to** | **Constrain** it |

> **The first row is where candidates trip.** An agent choosing `get_customer` over `lookup_order` from a set of **five** is a description problem. The same symptom from a set of **22** is not — the agent is past the point where description quality rescues selection, and rewriting all 22 leaves decision complexity exactly where it was. **Same symptom, different disease. Count the tools before you pick the remedy.**

### Consolidating near-duplicate tools

> A data platform server exposes **22 tools**: three query tools (one per source) and **19 transformations** — `pivot_table`, `calculate_percentile`, `normalise_currency`, and so on.

Splitting by role hands a transformation agent 19 tools — the original problem moved one level down. Instead, **collapse them: they share a shape (data in, an operation, data out).**

```json
{ "name": "transform_data",
  "description": "Apply a transformation to a dataset. Use transform_type to select the operation.",
  "input_schema": { "type": "object",
    "properties": {
      "dataset": { "type": "string" },
      "transform_type": { "type": "string",
        "enum": ["pivot", "percentile", "normalise_currency", "..."] },
      "options": { "type": "object" } },
    "required": ["dataset", "transform_type"] } }
```

**Twenty-two tools become four, and nothing is lost** — every transformation is still reachable, now as an enum value chosen *inside* one call rather than a tool to be found among nineteen near-identical descriptions. **Selection accuracy improves because the hard choice got smaller, not because the capability did.**

> **Consolidation and constraining pull in opposite directions, and the exam likes that tension.** Consolidation reduces *how many* tools the agent chooses between; constraining reduces *what any one tool can reach*. Collapsing 19 transformations grants no new powers, so it doesn't undo least privilege. Both can be right in the same system.

> ⚠️ **One fix that isn't a fix: moving tools onto a second MCP server.** Server boundaries are **invisible to the model** — the client hands it every tool from every connected server as one flat list. A 22-tool problem split across two servers is still a 22-tool problem.

### Scoped cross-role tools

> A synthesis agent constantly needs simple fact verification. Routing each one through the coordinator adds **2–3 round trips** and **40%+ latency**. Analysis shows **85% are simple lookups**.

**Fix: give the synthesis agent a scoped `verify_fact` tool for the simple case.** Complex verifications — multiple sources, cross-referencing, real judgement — still route through the coordinator. The 85% is handled locally; the 15% uses the full pipeline.

> This is the exam guide's **sample Question 9**. Distractors: remove the verification step entirely; cache at the coordinator; increase coordinator parallelism. All leave the round trip in place.

### Constrained alternatives — least privilege

Instead of giving a subagent `fetch_url` (which can fetch anything from anywhere), give it **`load_document`**, which validates document URLs only:

- Prevents misuse — the agent cannot fetch arbitrary URLs.
- Makes the purpose clearer — the description is specific, not generic.
- Reduces unintended side effects.

### Role-specific scoping in practice

| Agent | Tools |
|---|---|
| Web search | `search_web`, `fetch_page`, `extract_links`, `save_snippet` |
| Document analysis | `extract_metadata`, `extract_data_points`, `summarize_content`, `verify_claim` |
| Synthesis | `compile_report`, **`verify_fact` (scoped)**, `format_citation`, `assess_coverage` |
| Coordinator | `Agent` (formerly `Task`, spawns subagents), `review_output`, `request_revision` |

The coordinator runs the workflow **without holding any domain-specific tools itself**.

### The `tool_choice` parameter

| Value | Effect | Use when |
|---|---|---|
| `{"type": "auto"}` (default) | Claude decides whether to use a tool at all | Normal conversational agents |
| `{"type": "any"}` | Claude **must** call *some* tool, its choice which | You need structured output but the right tool depends on the input |
| `{"type": "tool", "name": "x"}` | Claude **must** call tool `x` | A specific step must happen first |
| `{"type": "none"}` | No tools may be called | Pure text turns |

**`tool_choice` applies to the turn you set it on.** This is the detail the exam tests.

### Forcing *some* tool: `"any"`

> An extraction pipeline has separate tools for invoices, contracts, and receipts. With `tool_choice: "auto"`, Claude sometimes returns conversational text instead of calling an extraction tool, breaking downstream parsing. The document type is **not known in advance**.

**Fix: `tool_choice: {"type": "any"}` with all three extraction tools defined.** Claude must call one of them — guaranteeing structured output — while still choosing whichever schema fits the document. One API call, no classification step.

The near-miss: *a preliminary classification call, then force the specific tool on a second call.* It also guarantees structured output, but costs an extra round trip to determine something the model can determine while extracting. When the question emphasizes that the type is unknown in advance, `"any"` is the answer; when a specific step must come first, `{"type": "tool"}` is.

### Forcing a *specific* tool, for one turn only

> A pipeline has `extract_metadata`, `lookup_citations`, and `verify_doi`. The user asks "extract the metadata and tell me how cited it is." The agent calls `lookup_citations` first, which fails because it needs a DOI that only `extract_metadata` can produce.

**Fix: set `tool_choice: {"type": "tool", "name": "extract_metadata"}` for *that turn*, then let subsequent turns run with `auto` so the enrichment tools can fire once the DOI exists.**

The over-correction distractor: force `extract_metadata` on **every** call. Then the enrichment tools can never run — you've fixed the ordering by removing the rest of the workflow. **The sequencing requirement applies to the first turn only; set the constraint, then release it.**

### Tool count degrades selection accuracy

> An agent system has **18 specialized tools** shared across all subagents. Subagents routinely call tools outside their specialization — the synthesis agent attempts web searches, the report generator tries document analysis.

**Cause: selection reliability falls as the candidate list grows.** Going from 4–5 focused tools per role to 18 shared tools pushes past the point where the model can reliably tell which tools belong to which role.

**Fix: give each subagent only the 4–5 tools relevant to its role.**

The distractor is "the tools consume too much context-window space." More tools do consume more tokens — but the observed symptom is agents **confidently calling the wrong tool**, which is a *selection-accuracy* problem, not truncation. Truncation would look like malformed calls or dropped content.

**Diagnostic:** confident-but-wrong = selection accuracy. Malformed or missing = truncation.

### Discovery-based availability at larger scale

At 50+ tools, restricting per-role isn't enough. The fix is dynamic availability: tools are hidden until a search/discovery tool surfaces them, so the model can only call what it has found. (Same scenario as 2.1 Problem 6 — the exam files it under both headings.)

| Catalog size | Approach |
|---|---|
| A handful | Define them all |
| ~10–20 across several roles | Restrict each subagent to its 4–5 relevant tools |
| 50+ | Discovery / deferred loading — hidden until surfaced by a search tool |

### Ordering is not a priority signal

Reordering entries in the `tools` array does **not** reliably control which tool Claude selects. If you need ordering enforced, use `tool_choice` or make the dependency structural (the second tool requires an ID only the first can produce).

### Exam traps for 2.3

- **Giving one agent 18 tools and expecting reliable selection.**
- **Rewriting descriptions when the toolkit is the problem** — past ~5 tools, decision complexity is the cause.
- **Splitting by role when the tools are variations on one job** — consolidate into a parameterised tool instead.
- **Moving tools to a second MCP server** — server boundaries are invisible to the model.
- **Routing every simple cross-role request through the coordinator** when a scoped tool would handle the common case.
- **Giving a subagent a generic `fetch_url`** when a constrained `load_document` would do.
- Using `"auto"` when structured output is required.
- Forcing a specific tool permanently instead of for the turn that needs it.
- Blaming context size for confident wrong-tool selection.
- Believing tool order in the array conveys priority.

---

## 2.4 MCP server integration

### The three MCP primitives

This is the most commonly confused part of the domain. Memorize the invocation model, not just the names.

| Primitive | Invoked by | What it is | Surfaced as |
|---|---|---|---|
| **Tools** | **The model** | Operations Claude can call during a loop | Regular tools in the tool list |
| **Resources** | **The application / browsed** | Contextual, browsable data — schemas, catalogs, file trees | Attachable context, not auto-called |
| **Prompts** | **The user** | Reusable templates / workflows | Slash commands: `/mcp__<server>__<prompt>` |

### MCP prompts become namespaced slash commands

> A custom MCP server provides DevOps workflow templates and exposes MCP prompts named `deploy_checklist` and `incident_response`. How do these become accessible?

**They appear as slash commands namespaced by server name: `/mcp__servername__deploy_checklist`,** with arguments passed after the command name.

The distractors blur the primitives:

- *Claude invokes them automatically like tools* — that is what **tools** are. Prompts are **user**-invoked.
- *They're automatically prepended as system context* — that is closer to what **resources** can do. Prompts are explicit invocations.

**Mnemonic: Tools = model calls them. Prompts = user calls them. Resources = things to look at.**

### All configured servers are available simultaneously

> An agent is configured with three MCP servers: git operations, Jira, and documentation search. The user asks: "create a branch for JIRA-123 and add documentation links to the ticket."

**Tools from all configured MCP servers are discovered at connection time and are all available simultaneously.** Claude freely combines a git tool, a Jira tool, and a docs tool within one request.

The distractor — *the system auto-selects "the most relevant server" and loads only its tools* — would make this cross-system request impossible. There is no server-selection step, and no routing by name prefix. The namespacing in tool names is for disambiguation, not routing.

### Project vs. personal configuration scope

> A team wants a shared venue-lookup MCP server available to everyone, and an individual wants a personal experimental music-playlist server for their own testing.

| Server | Location | Why |
|---|---|---|
| Shared venue lookup | **`.mcp.json`** in the project, committed | Travels through version control to every teammate |
| Personal experimental | **`~/.claude.json`** | User-local; doesn't touch anyone else's environment |

The distractor puts both in the project `.mcp.json`, exposing an untested experimental server to the entire team. **Scope follows audience** — the same principle as CLAUDE.md in 3.1.

> **Exam answer: two levels, project vs. user.** Current MCP docs define *three* scopes — `local` (the default), `project`, and `user`, selected with `-s` / `--scope`, with `~/.claude.json` holding both the local and user entries. Answer the two-level split.

### Entry shape, and a configuration error worth recognizing

```json
{
  "mcpServers": {
    "github":     { "type": "http", "url": "https://api.githubcopilot.com/mcp/" },
    "filesystem": { "command": "npx",
                    "args": ["-y", "@modelcontextprotocol/server-filesystem",
                             "${WORKSPACE_ROOT:-.}"] }
  }
}
```

| Shape | Declares | Transport |
|---|---|---|
| **Remote** | `"type": "http"` + `url` | HTTP |
| **Local** | `command` + `args` | stdio |
| ⚠️ **`url` with no `type`** | **A configuration error** | Claude Code reads it as a stdio server, skips it, and tells you to add the `type` |

### Environment variable expansion

`.mcp.json` supports `${VARIABLE_NAME}` expansion. **This is how you keep credentials out of version control while still sharing server configuration with the team.**

```json
{ "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}", "DATABASE_URL": "${DATABASE_URL}" } }
```

The file references variable *names*, never values:

- The configuration is safe to commit.
- Each developer authenticates with their own credentials.
- Token rotation needs no config change.
- No secrets enter repository history.

**Second form: `${VAR:-default}`** expands to `VAR` when set and falls back to `default` when not — for machine-specific paths with a sensible fallback, as in `${WORKSPACE_ROOT:-.}` above.

> **Gotcha:** an unset variable with no default does **not** halt the rest of the configuration. Claude Code warns and carries on, so don't rely on a missing token failing loudly.

**Trap: committing credentials directly into `.mcp.json`.** Credentials in version control are a security finding; expansion is the documented fix.

### Resources solve blind exploration

> A productivity agent connects to three MCP servers (issue tracker, documentation wiki, database explorer). Complex cross-system questions require **8–10 sequential exploratory calls** because the agent has no visibility into what each server contains. It exhausts context before answering.

**Fix: expose each server's content catalog as MCP resources** — issue summaries, documentation hierarchies, database schemas. With the map available upfront, the agent plans a targeted path instead of probing blindly.

This is the clearest illustration of what resources are *for*: **tools let you act; resources let you know what's there before you act.**

The distractor — *add a `prepare_investigation` tool to each server* — keeps the agent reactive. It still has to query to discover what exists; you've just wrapped the probing in a nicer name.

### Make MCP tool descriptions beat the built-ins

> An MCP server exposes `analyze_dependencies`, described as *"Analyzes dependency graph."* The agent keeps using the built-in **Grep** (*"Search file contents... Returns matching lines"*) to look for imports, even when users explicitly ask about "code dependencies."

**Fix: expand the MCP tool's description to spell out its unique capabilities and outputs** — e.g. *"Builds a dependency graph showing direct imports, transitive dependencies, and circular dependency cycles. Use for questions about how modules depend on each other."*

"Analyzes dependency graph" is four words. Grep's description is concrete about what it does and what it returns. At selection time the model picks the tool whose description better matches the request — so make yours specific.

- *Remove Grep from the tool list* — eliminates the overlap but breaks every other workflow that legitimately needs content search.
- *Add routing instructions to the system prompt* — lives outside the tool definition and doesn't travel with the server.

### Reuse before you build

> A team wants the agent to read Jira tickets during code review — descriptions, acceptance criteria, recent comments. Options: shell out to `curl` via Bash, build a custom MCP server, export tickets to Markdown, or integrate an existing Jira MCP server.

**Integrate the existing, purpose-built Jira MCP server.** It already exposes tickets, comments, and metadata through discoverable tool interfaces, with no integration code to build or maintain.

Building a custom server is a reasonable *fallback* when nothing suitable exists — the exam offers it as the trap for people who assume custom is more "correct." Bash/curl gives you no schema, no discoverability, and credentials in command lines. Markdown exports go stale immediately.

### Exam traps for 2.4

- Mixing up tools (model-invoked) / prompts (user-invoked) / resources (browsable data).
- Believing MCP servers need manual selection or route by name prefix.
- Putting experimental personal servers in the committed project config.
- Building a custom server when a suitable one already exists.
- Fixing MCP tool selection in the system prompt instead of the tool description.

---

## 2.5 Built-in tools

Only 3 questions, but they are free points.

| Tool | Searches / does | Use when |
|---|---|---|
| **Grep** | File **contents** | You know text that appears *inside* a file |
| **Glob** | File **paths** | You know a filename or path pattern |
| **Read** | Loads a file into context | Before editing; to understand matched results |
| **Edit** | Replaces a unique `old_string` | Targeted change with unambiguous surrounding text |
| **Write** | Writes a whole file | Creating, or rewriting when no unique anchor exists |
| **Bash** | Shell commands | Builds, tests, git — not file editing |

### Grep for content, Glob for paths

> Security scan: locate every occurrence of the dangerous `eval()` function across a large codebase.

**Grep for `"eval("`.** Glob matches *file paths* — it can find files named `eval.js` but cannot find a call site inside `utils/parser.js`.

> An engineer sees an unfamiliar production error: `SYNC_CONFLICT: entity version mismatch detected`. The codebase contains **12 services**. Which one emits it?

**Grep for a distinctive literal from the message** (`"SYNC_CONFLICT"` or `"entity version mismatch"`), then **Read** the matching files for context. Error strings are almost always unique enough to pinpoint the exact file in one search.

- *Glob the error-handling directories, then read each file* — many turns, lots of irrelevant reading.
- *Read the README, then work through service directories systematically* — the slowest possible approach to a problem with a one-command answer.

**Pattern: search by the most distinctive literal string available, then read only the hits.**

### Edit vs. Read + Write

> You want to insert a new function into a file, but `Edit`'s `old_string` can't find unique text — the file has repetitive docstrings, variable names, and structural patterns.

**Fix: `Read` the file, construct the change against the actual content, then `Write` the updated file.**

- *Use `Edit` with a 30+ line `old_string` to force uniqueness* — fragile; any whitespace or formatting discrepancy fails the match, and it's unreadable.
- *Use `Bash` with a heredoc to append the function* — likely lands it in the wrong scope and bypasses the file's structure.

**Also: always `Read` before `Write` or `Edit`.** You cannot construct a correct edit against content you haven't seen.

---

## The six sentences that answer most of Domain 2

1. **Fix the interface, not the wording.** Fabricated parameters, wrong tool selection, and unusable output are schema and description problems.
2. **Prefer impossible over discouraged.** Splitting a tool, requiring a token, or hiding a tool until discovered beats examples that describe correct usage.
3. **Return IDs and fields, never prose.** A response the next turn can't address is a broken response.
4. **Match the recovery to the error.** Transient → retry inside the tool. Permanent/business → surface with a category, a reason, and what to try next.
5. **Protocol errors are about the call; tool result errors are about the outcome.** A downstream 404 or 503 is a tool result error.
6. **Tools are model-invoked, prompts are user-invoked, resources are browsable.** All configured MCP servers' tools are available at once.

## Study order within this domain

1. **2.1 interface design** — 20 questions; learn the split/lookup/token/atomic patterns and the "fixes only two of three symptoms" trap.
2. **2.2 error responses** — 16 questions; memorize the structured payload shape and the protocol-vs-result boundary.
3. **2.4 MCP** — 8 questions, under-represented relative to the 18% domain weight; study the three primitives and config scoping directly from the lesson.
4. **2.3 tool_choice** — 6 questions; small, mechanical, memorizable.
5. **2.5 built-in tools** — 3 questions. Grep contents, Glob paths, Read before Write. Free points.
6. **[2.0 cross-topic questions](../../categorized_exam_dump/domain-2-tool-design-mcp-integration/2.0-cross-topic-and-domain-level.md)** — eight authored scenarios, several of which present interface problems in workflow or error-handling clothing.

## Links

[Domain 2](https://claudecertificationguide.com/learn/2-tool-design-mcp) ·
[2.1 Tool schema design](https://claudecertificationguide.com/learn/2-tool-design-mcp/2-1-tool-schema-design) ·
[2.2 Structured error responses](https://claudecertificationguide.com/learn/2-tool-design-mcp/2-2-structured-error-responses) ·
[2.3 Tool distribution & choice](https://claudecertificationguide.com/learn/2-tool-design-mcp/2-3-tool-distribution-choice) ·
[2.4 MCP server integration](https://claudecertificationguide.com/learn/2-tool-design-mcp/2-4-mcp-server-integration) ·
[2.5 Built-in tools](https://claudecertificationguide.com/learn/2-tool-design-mcp/2-5-built-in-tools)
