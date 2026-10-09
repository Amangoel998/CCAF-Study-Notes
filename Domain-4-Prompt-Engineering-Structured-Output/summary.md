# Domain 4 — Prompt Engineering & Structured Output

**Blueprint weight: 20% (~12 of 60 questions) · 57 questions in the bank, ~33 distinct concepts**

> The highest duplication in the bank — nearly every numbered question has a screenshot twin, and several have two. Fifty-seven questions reduce to roughly thirty-three concepts. See [4.0 Cross-Topic & Domain-Level](../../categorized_exam_dump/domain-4-prompt-engineering-structured-output/4.0-cross-topic-and-domain-level.md) for the clusters.

> **The one sentence that governs this domain:** the **schema** controls *shape*, **examples** control *judgment*, **validation** controls *meaning*. Most wrong answers apply one of these to a problem that belongs to another.

Before anything else, internalize this triage. Nearly every D4 question is a disguised version of "which layer is broken?"

| Symptom | Broken layer | Fix |
|---|---|---|
| Malformed JSON, parse failures, prose wrapped around the object | **Shape** | Tool use with an input schema |
| Schema-valid output with invented, inconsistent, or wrongly-granular values | **Judgment** | Few-shot examples; optional/nullable fields; enums with escape hatches |
| Values that pass the schema but violate a business rule | **Meaning** | Semantic validation, cross-field checks, retry with the specific error |
| Correct early in a session, degraded later | **Attention** | Reinforcement, versioning, splitting the work |

And the single most important consequence: **"passed schema validation" ≠ "correct."** A schema guarantees that `attendee_count` is an integer. It does not guarantee the number appeared in the document.

---

## 4.1 System prompts with explicit criteria

10 questions. Three themes: **what belongs in a system prompt**, **how to write criteria that generalize**, and **what to do when adherence drifts**.

### The single biggest mistake: vague instructions

The guide opens with this, and the exam uses these exact phrases as distractors:

> *"Be conservative."* · *"Only report high-confidence findings."* · *"Use your best judgement."*

None gives the model an actionable decision boundary. They **sound** reasonable, which is precisely why they are offered.

| | Prompt |
|---|---|
| ❌ **Wrong** | "Review this code. Be conservative. Only report high-confidence findings." |
| ✅ **Correct** | "Flag comments only when claimed behaviour contradicts actual code behaviour. Report bugs and security vulnerabilities. Skip minor style preferences and local patterns." |

The correct version supplies **explicit categorical criteria**: what to report, what to skip, and a specific trigger condition. "Conservative" means different things in different contexts, and "high-confidence" is a threshold the model cannot calibrate.

**Define both halves — what to flag *and* what to skip.** Omitting the skip list is why precision stays low.

### Severity calibration needs code examples, not prose

| Approach | Example |
|---|---|
| ❌ **Prose** (insufficient) | *"Critical: issues that could cause system failures or data loss. Minor: issues affecting readability but not functionality."* |
| ✅ **Code example** (correct) | **Critical** — unsanitised input in a SQL query: `query = f"SELECT * FROM users WHERE id = {user_input}"`<br>**Minor** — inconsistent naming: `userName` vs `user_name` in one module |

Prose forces the model to interpret what *"could cause system failures"* means. A code example removes the ambiguity entirely, and classification becomes stable across invocations.

### The false-positive trust problem

> A review pipeline has a **40% false-positive rate on one category**. Developers stop reading **all** categories — including a security category running at 98% accuracy.

**Trust is not category-specific. It bleeds across the whole output.**

**Fix — and it feels backwards: temporarily disable the high-false-positive category while you rework its prompt.** Trust in the categories that already work returns immediately. Iterate on the broken category with concrete code examples and switch it back on only once precision improves.

You are not abandoning the category. You are putting **system-wide trust ahead of category completeness.**

Why the alternatives fail:

- *Add "only report high-confidence documentation issues"* — the vague instruction that caused the problem.
- *Raise temperature and keep findings that recur across runs* — expensive, and ensemble agreement is not accuracy.
- *A second model pass that discards unverifiable findings* — adds cost and latency without fixing the criteria.

### Why confidence-based filtering fails

"Only report high-confidence findings" is a frequent tempting answer. **LLM self-reported confidence is poorly calibrated** — the model is often sure about wrong findings and hesitant about right ones.

> **The hierarchy: explicit criteria first, confidence-based routing second. Never skip the first step.**

Confidence scores earn their keep in **routing** (sending low-confidence findings to human review — 4.6 and 5.5). They are not a substitute for defining what counts as a valid finding in the first place.

### What belongs in the system prompt, and why placement matters

> A music discovery assistant must maintain an enthusiastic tone, explain its reasoning for recommendations, and ask clarifying questions — across all interactions.

**Behavioral rules belong in the system prompt.** It is sent with every API call and carries the most authority over the whole conversation.

The distractors are a ranking exercise in positional authority:

| Placement | Authority |
|---|---|
| **System prompt** | Highest — present every call, applies conversation-wide |
| First assistant message | Weaker — it's one turn among many, easily diluted by later context |
| Prefixed to each user message | Weak — reads as part of the user's request; fights conversation content |
| Environment variables | None — outside the API payload entirely; Claude never sees them |

### Injecting runtime state

> During an order-tracking conversation, an external webhook reports that the package has shipped. The user will send a follow-up message soon, and the assistant should naturally incorporate the new status.

**Fix: add the current shipping status to the system prompt before the next API call.**

The state is now in context, so when the user writes, the assistant weaves it in naturally — no interruption required.

- *Send the update as a synthetic user message* — interrupts the conversation with a message the user didn't send, and misattributes system information to the user.
- *Prefix it to the next user message* — works, but mixes system state into user content; less clean and less authoritative.
- *Configure a tool to call `get_order_status` before every response* — latency and cost on every turn to retrieve data you already have pushed to you.

**Pattern: changing state goes in the system prompt, refreshed before the next call.** This is the D4 twin of Domain 1's structured state object and Domain 5's persistent facts block.

### Write principles that generalize, not conditional branches

> A fitness coaching assistant has detailed conditional logic for beginner/intermediate/advanced users, keyed on explicit terms like "progressive overload" or "superset." When users don't state their level, it defaults to overly detailed responses — missing implicit expertise signals in their vocabulary.

**Fix: replace the narrow conditionals with a general adaptation principle** — *"Adapt explanation depth to match the user's apparent expertise, mirroring the terminology they use"* — keeping only safety-critical conditionals explicit.

Enumerated triggers can only fire on cases the author anticipated. A principle generalizes to signals nobody listed.

- *Add more conditional branches* — the next unanticipated signal still falls through. You can't enumerate a language.
- *A pre-conversation intake form* — adds friction and still ignores the contextual cues already present in the user's writing.
- *Ask a clarifying question on the first message* — same friction, same failure to use available evidence.

**Note the scope qualifier:** safety-critical conditionals stay explicit. Principles for judgment; rules for safety.

### When the criterion is a checklist, make it enumerable

> A code review prompt for test-coverage analysis correctly flags functions with **no** tests, but misses untested conditional branches and error-handling paths **inside** tested functions.

**Fix: add explicit instructions requiring Claude to enumerate every conditional branch and exception path, then verify each has a corresponding test assertion.**

"Check test coverage" is satisfied by function-level checking. The prompt never specified the granularity. Making the procedure explicit — *enumerate, then verify each* — forces branch-level inspection.

- *Interleave implementation and tests in the prompt* — a layout change; the checking instruction is still missing.
- *A two-pass pipeline* — would work, but the question explicitly rules out added complexity.
- *Few-shot examples alone* — see the 4.2 companion question; examples show the output, instructions direct the process. (The exam asks this scenario from both angles, so read which one the question is after.)

**Pattern: when a criterion has internal structure, say "enumerate X, then verify each."** This is the generalizable version of "list every branch, then check it has a test."

### Statelessness, again

> In a vocabulary tutoring conversation, Claude appears to forget words discussed earlier. Asked "can you quiz me on those words?", it responds as if nothing was discussed. **This fails from the very first test, not after a long conversation.**

**Cause: the Messages API is stateless.** Unless the application includes all prior turns in the `messages` array on each request, Claude has no record of them.

- *"The system prompt needs explicit memory instructions"* — no instruction can recover a message that wasn't sent.
- *"The context window filled up"* — would appear only after a long conversation. **"Fails from the start" rules this out.**
- *"Enable conversation persistence via a session ID parameter"* — no such API mechanism exists.

**Always check whether the scenario says the failure is immediate or gradual.** Immediate = missing data. Gradual = attention/context pressure.

> **Filing note:** the bank now files this under **5.1 Context Window Management**, since the concept is conversation-state management rather than prompt design. It stays here because the stem reads like a prompt problem and the contrast with the drift scenarios below is the whole point — immediate failure and gradual failure sit side by side in this subtopic, and you need both to tell them apart.

### Drift: when adherence degrades over turns

Two scenarios, same cause, different remedies.

> **Scenario A.** A home renovation assistant follows its guidelines (ask about budget, suggest multiple alternatives, confirm timeline) consistently through turns 1–4, but by turn 7 gives generic advice without asking about budget or timeline. The conversation is **2,500 tokens of a 200,000-token window**.

> **Scenario B.** Claude follows behavioral guidelines for turns 1–10, but by turns 25–30 deviates on tone, formatting, and information restrictions. The conversation is **30,000 of 200,000 tokens**.

Both scenarios go out of their way to tell you the window is nowhere near full. That is deliberate: **this is not context exhaustion.** As conversation turns accumulate, their combined content competes with the system prompt for the model's attention. Influence is diluted, gradually.

The wrong explanations:

- *"The system prompt is only sent with the first request."* If true, adherence would collapse at turn 2 — not erode gradually through turn 7.
- *"The model's attention naturally weakens on system prompts over time."* Anthropomorphic non-explanation. The mechanism is competition from accumulated context.

**The fix (Scenario B): insert user-role messages that reinforce the critical guidelines at natural breakpoints** — especially before complex requests.

- *Post-response validation and regeneration* — doubles cost and latency to repair what reinforcement prevents.
- *Automatically start a new conversation after 20 turns* — fragments the session and loses continuity; operational complexity for a prompt-level problem.
- *Move the guidelines from the system prompt into the first user message* — strictly weaker positioning. Moving in the wrong direction.

**Related (4.2): if the guidelines themselves are verbose prose, replacing them with few-shot examples makes them more compact *and* more concrete, improving the signal-to-context ratio.** Reinforce when the guidelines are already good; exemplify when they're abstract.

### Versioning prompts so old conversations keep their contract

> After deploying an improved system prompt, users with **multi-week, multi-session** conversations report contradictions with earlier statements and a noticeably different communication style. New users report no issues.

**Cause: an existing conversation's history was produced under the old behavioral contract, and the new prompt now governs it.** The mismatch surfaces as self-contradiction.

**Fix: version the system prompts and bind each conversation to the version it started under. Apply updates only to new conversations.**

- *Regenerate or replace the stored conversation histories* — rewriting what was actually said. A data-integrity violation, and users will notice.
- *Add a transition message explaining the update* — documents the inconsistency without resolving it.
- *Add instructions to the new prompt telling it to stay consistent with prior statements* — the new prompt still conflicts with history built under different rules; you're asking it to honour two contracts at once.

### Ambiguity: assume, state, invite correction

> An AI tutor receives ambiguous requests like "Can you help me with the report?" and responds with **three or more clarifying questions** (Which report? What kind of help? What's the deadline?). **40% of conversations are abandoned** after these multi-question responses.

**Fix: instruct the assistant to make reasonable assumptions from available context, state those assumptions explicitly, and offer to adjust if the interpretation is wrong.**

> "I'll assume you mean the Q3 analysis report you mentioned earlier and that you want help structuring the findings section — let me know if you meant something else."

Momentum is preserved and the user retains control.

- *Limit it to one clarifying question per turn* — stretches the interrogation across several turns. Same friction, slower.
- *Route high-ambiguity requests through a preprocessing classifier* — infrastructure that still ends in a question.
- *Use a lookup table of predefined defaults* — assumes **silently**, which removes user agency. The explicit statement of the assumption is the load-bearing part.

**This is the central ambiguity pattern and it recurs in 5.2: assume → state → invite correction.** Ask only when the options lead to materially different actions or the step is irreversible.

### Controlling openings with a partial assistant prefix

> Responses repeatedly begin with stock phrases — "Certainly!", "I'd be happy to help!" — deep into conversations. Abandonment exceeds **40%**; users want more natural, varied flow.

**Fix: append a partial assistant message with a direct opening that Claude continues from.** Prefilling the start of the turn removes the slot where the stock greeting would have gone.

- *Post-process to strip the phrases* — reactive; often leaves awkward fragments; doesn't change generation.
- *System prompt instructions listing phrases to avoid* — banning specific phrases doesn't teach variety; the model substitutes different stock phrases.
- *Lower the temperature* — **backwards.** Lower temperature makes the model *more* likely to pick the single highest-probability opening — more repetitive, not less.

### Exam traps for 4.1

- **Choosing "be conservative" or "only report high-confidence findings" as a prompt improvement.** Vague instructions don't improve precision.
- Using a confidence threshold to fix false positives — confidence is poorly calibrated; define criteria first.
- Describing severity in prose when concrete code examples are available.
- **Keeping a high-false-positive category enabled** while iterating on it, when disabling it would restore system-wide trust.
- Reading "fails from the start" as a context-window problem (it's missing history).
- Reading gradual drift as context exhaustion when the token counts are explicitly low.
- Enumerating more conditions instead of stating a general principle.
- Moving behavioral rules out of the system prompt to a weaker position.
- Asking a pile of clarifying questions, or silently defaulting without stating the assumption.
- Lowering temperature to increase variety.

---

## 4.2 Few-shot prompting

10 questions. The constant: **use examples when the failure is a judgment call that prose cannot pin down.**

> **The guide states this as a direct exam principle:** when output is inconsistent, few-shot examples are the **first** tool to reach for — not more instructions, not confidence thresholds, not temperature. The exam presents scenarios where detailed instructions already produce inconsistent results and tests whether you choose "add more instructions" or "add examples." **It is almost always the latter.**

### The three triggers

Deploy few-shot examples when you see one of these, and only these:

| # | Trigger | What it looks like |
|---|---|---|
| **1** | **Detailed instructions still produce inconsistent formatting** | A thorough prompt specifies the format, but output arrives as a bulleted list one run, a table the next, prose the third. More instructions will not fix this |
| **2** | **Inconsistent judgement on ambiguous cases** | The same pattern rated "critical" in one file and "minor" in another; the same request routed to different tools depending on phrasing |
| **3** | **Empty/null fields for information that *exists*** | The data is present but in an unexpected form — embedded in narrative rather than a table, or split across paragraphs |

Note trigger 3 carefully: it is the mirror image of the 4.3 fabrication case. **Information present but unrecognized → examples. Information genuinely absent → nullable schema fields.**

### How to construct them — the rules are tight

**Use 2–4 targeted examples.** Fewer than 2 doesn't establish a pattern; more than 4 wastes tokens without proportional benefit. Point them at the specific scenarios that are failing.

**Every example must show *reasoning*, not just input→output.**

```
Input: "check my order #12345"
Selected tool: lookup_order
Reasoning: The user provides an order number (#12345), indicating they want
  order-specific information. Even though this could be read as a general
  customer query, the specific identifier makes lookup_order correct over
  get_customer.
```

Without the reasoning the model learns only *"queries mentioning order numbers go to `lookup_order`."* With it, the model learns the **general principle** — specific identifiers route to specific lookup tools — and generalizes to novel phrasing.

> **Trap: thinking examples only teach literal pattern-matching.** They do, *if* you omit the reasoning. Reasoning is what converts an example into a transferable rule.

**Cover the failing scenarios.** If extraction works on tables and fails on narrative, the examples must show narrative extraction. If severity is inconsistent on one pattern, show that pattern classified at different levels *with the reasoning for each*.

### Two side effects worth knowing

**Hallucination reduction.** Showing extraction from varied structures — inline citations vs. bibliographies, narrative vs. tables — teaches the model to handle structural variety *without inventing data*. Without examples it often nails the table and either empties or fabricates the narrative section.

**False-positive reduction.** In review tasks, examples pull double duty by showing both **what to flag and what to ignore**. An example classifying a benign shadowing pattern as *minor*, with reasoning about why it's a style preference rather than a defect, teaches the boundary without suppressing genuine findings.

### Few-shot vs. other techniques

The exam tests whether you reach for examples when something else is the right tool:

| Symptom | Correct technique |
|---|---|
| Inconsistent output formatting | **Few-shot examples** |
| Malformed JSON | `tool_use` with a JSON schema |
| Fabricated values for missing fields | Optional/nullable schema fields |
| Wrong tool selection | **Better tool descriptions first**, then few-shot |
| Model misses information in narrative text | **Few-shot examples** showing narrative extraction |
| Extracted sum doesn't match the total | Validation–retry loop |

Examples teach boundaries that are easy to demonstrate and hard to describe — what counts as "explicitly stated," whether to split a compound phrase, what granularity an entry should have, which code patterns are acceptable in *your* deployment context.

### The recurring distractor set

In nearly every 4.2 question the wrong answers come from the same four families:

| Family | Example | Why it loses |
|---|---|---|
| **Post-processing** | Regex normalization, keyword filtering, canonical taxonomy mapping | Repairs output; doesn't teach the judgment. Brittle across formats |
| **Schema change** | Make the field required / add a metadata enum | Changes the container, not the decision about what goes in it |
| **Model/parameter change** | Upgrade the tier, set temperature 0, fine-tune | Assumes a capability gap when the real gap is instruction clarity |
| **More prose** | "Be conservative", "consider intentional patterns", a longer written spec | Abstract where the problem is concrete |

Learn to spot these and the subtopic becomes mechanical.

### Varied source layouts

> A citation and methodology extraction pipeline produces valid JSON, but in **5%** of cases the required `citations` and `methodology` fields come back **empty — even though the information is present** in the source. The documents vary: inline citations vs. bibliographies; methodology in a dedicated section vs. embedded in the introduction.

**Fix: few-shot examples showing extractions from documents with each of these varied structures**, demonstrating how to recognize citations in different formats and locate methodology across section types.

The model isn't failing to format — it's failing to *recognize* the content when it appears in an unfamiliar layout. Examples teach the transfer rule.

- *Regex post-processing* — brittle across formats, and silently misses valid citations.
- *Make the fields optional* — accepts the failure instead of fixing it. (Contrast 4.3, where making a field optional is correct **because the data genuinely isn't there**. Here it is there. Same mechanism, opposite diagnosis — read the scenario carefully.)
- *Retry logic* — retrying without new guidance reproduces the same miss.

### Normalization conventions

> Product description extraction for a materials field produces `"Cotton/Polyester mix"`, `"cotton blend"`, or omits the field entirely — even when the material is clearly stated.

**Fix: few-shot examples with complete input→output pairs showing the standardized format and demonstrating that the field must be populated when evidence exists.**

The model has no way to guess your house style from a field name.

- *Temperature 0* — makes it deterministic, not correct. It will consistently produce one arbitrary format.
- *Upgrade the model tier* — assumes capability; the issue is an unstated convention.
- *Make the field required* — forces *a* value (possibly fabricated) without teaching the format.

### Compound phrases and entry granularity

> Skills extraction from résumés has three problems: (1) compound phrases like "Python and SQL" are split inconsistently, (2) implied-but-unstated skills appear in the output, (3) similar documents produce wildly different array lengths — 5–10 entries vs. 40+. The prompt says only *"Extract all skills mentioned."*

**Fix: few-shot examples demonstrating compound-phrase handling, explicit-mention criteria (reject implied skills), and appropriate entry granularity.**

All three are judgment calls. "Extract all skills mentioned" doesn't define *mentioned*, doesn't say whether "Python and SQL" is one skill or two, and gives no sense of granularity.

- *A numeric constraint ("extract 10–20 skills max")* — caps length without resolving split/combine decisions or the implied-skills problem. **Fixes one of three symptoms.**
- *Post-extraction normalization against a canonical taxonomy* — cleans up afterwards; the initial judgment is still inconsistent, and implied skills get normalized into legitimate-looking entries.
- *Enrich the schema with confidence and source metadata* — captures uncertainty; doesn't establish the consistency rules.

### Verbatim vs. normalized values

> A recipe extraction system has accumulated **847 human corrections over 3–6 weeks**. Analysis shows **23%** of corrections involve informal measurements — "a handful", "a splash" — where the model either invents a specific amount or leaves the field empty, instead of extracting the phrase verbatim.

**Fix: few-shot examples demonstrating verbatim extraction of informal measurements.**

The corrections data has already told you exactly what the rule should be. This is a specific, well-understood, narrow pattern — precisely what examples are for.

- *Post-processing pattern matching* — brittle; informal measurements are open-ended ("a dash", "to taste", "a good glug").
- *Add a `measurement_type` enum to the schema* — metadata about a value the model still extracts wrongly.
- *Fine-tune on all 847 corrections* — **the key trap.** Fine-tuning is heavy machinery for a narrow pattern a handful of examples resolves. Cost, latency, and a model to maintain, for something a prompt change fixes.

**Rule: never fine-tune for a pattern a few examples would fix.**

### Context-dependent judgment: acceptable vs. genuine findings

> An automated code review has a **35% false-positive rate**: style suggestions contradicting team conventions, security warnings for patterns that are safe in this deployment context, and performance suggestions that would degrade specific use cases. The model must generalize to **novel code patterns it hasn't seen**.

**Fix: few-shot examples with annotated code snippets distinguishing acceptable patterns from genuine issues in each category.**

"Safe in this deployment context" cannot be looked up — it's local knowledge. Annotated examples transfer it and generalize to new code.

- *Post-processing with keyword filtering* — context-dependent judgment can't be expressed as keywords; you'd suppress real findings too.
- *Instructions like "be conservative" or "consider that patterns may be intentional"* — no concrete criteria; also risks suppressing genuine findings, which is the 4.6 failure mode.
- *A comprehensive written specification* — describes without demonstrating; subtle distinctions are exactly what prose loses.

### Examples as a compactness win

> A conversational tutor has a **2,800-token** system prompt covering teaching methodology and detailed written instructions for adapting to proficiency level. In conversations beyond **12 turns (~4,000 tokens of history)**, it ignores the proficiency guidelines and defaults to intermediate-level explanations.

**Fix: replace the verbose written proficiency guidelines with few-shot examples showing appropriate responses at each level** — concrete differences in vocabulary, complexity, and explanation depth.

Two gains at once: examples are more concrete *and* more compact, improving the signal-to-context ratio that was losing to accumulated history.

- *Inject condensed reminders every 4–5 turns* — overhead, and it re-injects the same abstract prose that wasn't working.
- *Restructure the system prompt, moving proficiency rules to the final section* — better organization; the information-to-context ratio is unchanged.
- *A separate API call to evaluate the required difficulty level* — cost and latency to classify, with no improvement to the response itself.

### Examples vs. instructions (the branch-coverage pair)

The branch-coverage scenario appears in both 4.1 and 4.2, with different correct answers:

| Framing | Answer |
|---|---|
| "The prompt never directs the model to check at branch level" | **Explicit enumeration instructions** (4.1) |
| "The model doesn't know what the output should look like" | **Few-shot examples** pairing uncovered-branch code with the review comment naming the missing test case (4.2) |

**Instructions direct the process; examples demonstrate the product.** When both are offered, pick based on which the scenario says is missing. When only one is offered, take it. In either framing, the multi-pass-pipeline distractor loses because the question says to avoid added complexity.

### Exam traps for 4.2

- **Choosing "add more detailed instructions" when detailed instructions already exist** and output is still inconsistent.
- Omitting reasoning from examples, reducing them to surface pattern-matching.
- Using fewer than 2 or more than 4 examples.
- Using a confidence threshold to fix inconsistent judgement calls.
- Choosing post-processing when the model needs to learn the rule.
- Choosing fine-tuning for a narrow, well-characterized pattern.
- Choosing a numeric cap that addresses only one of several symptoms.
- Choosing temperature 0 or a bigger model for a convention problem.
- Pre-processing the input into a uniform shape instead of teaching the model to read varied shapes.

---

## 4.3 Structured output with tool use

The largest subtopic in the domain (17 questions, ~9 distinct concepts), and the one with the most memorizable patterns.

### The reliability hierarchy — commit this to memory

| Rank | Approach | Guarantee |
|---|---|---|
| **1** | **`tool_use` with a JSON schema** | Eliminates JSON **syntax** errors entirely |
| **2** | Prompt-based JSON | None. Will periodically produce unparseable output in production |

The tool's schema constrains the **shape** of what Claude returns — no missing brackets, trailing commas, or unquoted keys. The separate `tool_choice` parameter is what forces the model to call the tool **at all**. Keep those two roles distinct.

> **Newer API controls the guide predates:** `strict: true` on a tool definition, and `output_config.format`, which constrains the response itself rather than a tool call. **On the exam, answer with the tool_use-over-prompt-based hierarchy.**

### `tool_choice`: the three modes

| Mode | Behaviour | Use when |
|---|---|---|
| **`"auto"`** (default) | Model decides whether to call a tool **or return text** | It legitimately needs to respond conversationally |
| **`"any"`** | Model **must** call a tool, but chooses which | Multiple extraction schemas and the **document type is unknown**. Guaranteed structured output, flexible selection |
| **`{"type": "tool", "name": "..."}`** | Model **must** call that specific tool | Forcing a mandatory first step — e.g. metadata extraction before enrichment |

> The current API has a fourth mode, `{"type": "none"}`, which blocks tool calls and is the default when you pass no tools. **The guide names three — answer three.**

**`tool_choice` applies per *request*, not per conversation.** Once the forced call returns, send the next request with `auto` (or omit the parameter) — otherwise the model is obliged to call the same tool again and **you loop**.

**Trap: confusing `auto` with `any`.** `auto` permits a text response, so it gives **no guarantee** of structured output. For guaranteed structure with an unknown document type, use `any`.

### What `tool_use` does *not* prevent

This is where the exam gets sneaky. A schema eliminates syntax errors and nothing else. Three semantic failures survive it:

| Failure | Example |
|---|---|
| **Sum discrepancies** | Line items that don't sum to the stated total |
| **Field placement errors** | A date in an amount field — both are strings, so the schema is satisfied |
| **Fabrication** | The model invents a value for a required field the source doesn't contain |

**The schema guarantees structure. It does not guarantee correctness.** Semantic validation is a separate layer (4.4).

### Tool use is how you guarantee shape

> A system extracts event details (title, date, time, location, attendees) from calendar invitations into JSON. Downstream systems reject malformed or non-conformant JSON.

**Fix: define a tool whose input schema matches the target structure, and read the extraction from Claude's `tool_use` arguments.**

The reason this works is mechanical and worth stating precisely: **the structure is enforced at generation time, before any parsing happens.** Claude's tool-call arguments are constrained to the declared schema. Text-based JSON generation has no such constraint.

```json
{
  "name": "record_event",
  "input_schema": {
    "type": "object",
    "properties": {
      "title":    {"type": "string"},
      "date":     {"type": "string", "description": "ISO 8601 date, YYYY-MM-DD"},
      "location": {"type": ["string", "null"]},
      "attendees":{"type": "array", "items": {"type": "string"}}
    },
    "required": ["title", "date"]
  }
}
```

The distractors are all post-hoc repair:

- *Detailed JSON formatting instructions plus a template in the prompt* — text generation can still emit surrounding prose, trailing commentary, or edge-case malformations.
- *Pre-fill the response with an opening brace* — nudges the start; constrains nothing after it.
- *Regex extraction with retry* — brittle repair after the fact.
- *Two API calls — generate text, then format it as JSON* — the second call is still unconstrained text generation.

The identical pattern appears for résumé extraction. **If the question is about reliable schema conformance, the answer is tool use with an input schema.**

### The central lesson: required fields cause fabrication

This is the most counterintuitive and most-tested idea in D4.

> An extraction pipeline pulls event metadata (date, location, organizer, `attendee_count`) from news articles. The model fabricates plausible values — outputting `500` for `attendee_count` when the source says nothing about attendance.

**Fix: change fields that may not exist in the source from required to optional**, so the extractor can omit what isn't there.

A required field is an instruction: *produce a value here.* If the document doesn't supply one, the model produces a plausible one, because that is what satisfies the schema. **Making everything non-nullable makes fabrication worse, not better.**

- *Add a confidence score field* — **the signature trap.** A confidence field permits the fabricated value and attaches a number to it. The model rates its own guess, usually generously. Confidence fields never prevent fabrication.
- *Add semantic validation after extraction* — catches some fabrications after generation; doesn't prevent them, and a plausible fabrication often passes.
- *Write stronger instructions* — you cannot instruct your way around a schema demanding a value.

**Companion question:** when the schema already has all fields nullable but the model still fills them with plausible inventions, **add an explicit instruction to return `null` for any field not directly stated in the source.** Nullable *permits* null; the instruction *directs* it. Schema and instruction together.

### Vocabulary for "not stated" and "ambiguous"

> Product review extraction uses a schema with `rating` (int 1–5), `pros` (string array), `cons` (string array), `overall_sentiment` (enum: positive / negative / mixed). Brief reviews like *"Great product!"* cause fabricated specific pros and cons. Sarcastic reviews like *"Well that was... interesting"* get an arbitrary sentiment because there is no "unclear" option. About **20%** of the dataset is affected.

**Fix: allow empty arrays for `pros`/`cons`, and add `"unclear"` to the sentiment enum.**

Two distinct conditions need two distinct representations:

| Condition | Representation |
|---|---|
| Information not stated | Empty array, or `null` |
| Information stated but genuinely ambiguous | An explicit `"unclear"` enum value |

Every distractor fails by not giving the model a valid way to say "I don't know":

- *Add a confidence field 0.0–1.0* — metadata about an invented value. Still fabricates.
- *Make fields optional and add both `"neutral"` and `"unclear"`* — close, but `"neutral"` describes the *review's* sentiment, while `"unclear"` describes the *extractor's* certainty. Adding both invites confusion between a genuinely neutral review and an undeterminable one.

**Design rule: the schema must be able to express every true state of the world, including "absent" and "indeterminate."**

### Enums need an escape hatch

> A property-type extraction schema uses the enum `[house, apartment, condo, townhouse]`. After deployment, **8%** of extractions fail validation because listings mention studios, lofts, duplexes, mobile homes, tiny houses, and converted warehouses — **and new types appear regularly**.

**Fix: add `"other"` to the enum plus a separate `property_type_detail` string field populated when `"other"` is selected.**

You keep schema validation for the common cases *and* preserve the actual value for the long tail.

- *Keep expanding the enum* — reactive; "new types appear regularly" guarantees you'll fail again.
- *Change the field to a free-form string* — throws away validation entirely; you're back to inconsistent values.
- *Few-shot examples teaching the model to map unusual types to the closest enum value* — **lossy and silently wrong.** A converted warehouse recorded as "house" is a data error that nothing downstream can detect.

**Pattern: closed enum + `"other"` + a detail field.** Control where you can, capture where you can't.

### Amended documents: don't choose, record both

> Contract extraction encounters amendments — the original says "30-day payment terms", Amendment 1 says "45 days". The model inconsistently extracts one or the other, with no indication of which applies or which is current.

**Fix: redesign the schema so amendable fields hold multiple values, each tagged with source location and effective date.**

```json
{"payment_terms": [
  {"value": "30 days", "source": "§4.2 original", "effective_date": "2023-01-01"},
  {"value": "45 days", "source": "Amendment 1 §2", "effective_date": "2024-03-15"}
]}
```

The schema was built on a false assumption — that each field has exactly one value. Every distractor preserves that assumption:

- *Always extract the most recent amendment* — silently discards the original term, which has genuine audit and historical value ("what were the terms in Q2 2023?").
- *Pattern-match after extraction to flag amended contracts for review* — flags without preserving; a human re-reads the contract.
- *Preprocess the document to remove superseded sections* — destroys the audit trail before extraction even starts.

**When reality is multi-valued and temporal, the schema must be too.** Same instinct as Domain 5's provenance requirement.

### Normalize at extraction, not after

> Restaurant menu extraction must produce item names, descriptions, prices, and dietary tags. Menus format inconsistently — prices as `"$12"` vs `"12.00"`, dietary info as icons vs text.

**Fix: define a strict output schema *and* include explicit format normalization rules in the prompt** — prices as a float, dietary tags as an array of enum values.

Both shape and representation become predictable at extraction time.

- *Extract as-is and normalize in post-processing* — moves the inconsistency downstream, where you have less context to resolve it (is `"12"` dollars or euros?).
- *Separate extraction calls per field* — multiplies cost and latency, and can produce cross-field inconsistency.
- *Multiple attempts with majority voting* — expensive, and three consistently-wrong formats still vote wrong.

### Expose discrepancies; never silently reconcile

> An invoice pipeline extracts line items, subtotals, tax, and grand totals. In **18%** of extractions the line items don't sum to the extracted grand total — some are OCR errors in the source, some are model errors. Downstream accounting systems reject mismatched records.

**Fix: extract `calculated_total` (the model's sum of the line items) alongside `stated_total` (the figure printed on the invoice), and flag records for human review when they differ.**

This works because the discrepancy is a *fact about the document* that you cannot resolve from the extraction alone. Surfacing it is the correct output.

- *A separate validation model to reconcile the two* — cost and complexity to guess which number is right.
- *Few-shot examples of consistent extractions* — may reduce model-side errors; cannot fix an OCR error in the source, and flags nothing.
- *Auto-adjust the line items proportionally to match the total* — **the worst option.** It manufactures agreement, destroys the only signal that something was wrong, and writes fabricated numbers into an accounting system.

**Principle: a cross-field consistency check should raise a flag, never perform a silent correction.**

### The 4.3 schema design checklist

| Need | Design |
|---|---|
| Guaranteed shape | Tool use with an input schema |
| A field that may be absent | Optional / nullable **plus** an explicit "return null if not stated" instruction |
| "Not stated" for a list | Empty array |
| "Stated but ambiguous" | An explicit `"unclear"` enum value |
| An open category set | Closed enum + `"other"` + a detail field |
| Values that change over time | Array of `{value, source, effective_date}` |
| Internal consistency | Separate `calculated_*` and `stated_*` fields; flag on mismatch |
| Format consistency | Normalization rules in the prompt, enforced at extraction |

### Exam traps for 4.3

- Adding a confidence field to stop fabrication.
- Making fields required "so they always get filled."
- Expanding an enum reactively instead of adding `"other"` + detail.
- Picking one value when the document legitimately contains several.
- Auto-correcting a discrepancy instead of flagging it.
- Choosing prompt-based JSON instructions over tool use.

---

## 4.4 Validation, retry, and feedback loops

Only 5 questions, but they encode a precise rule about **when retrying can and cannot help**.

### The retry message carries three things

| # | Element | Why |
|---|---|---|
| 1 | **The original document** | So the model can re-examine the source |
| 2 | **The failed extraction** | So it can see what it produced |
| 3 | **The specific validation error** | So it knows exactly what went wrong |

```
Original document:
{document}

Your extraction:
{failed_json}

Validation error: Line items sum to £450 but stated_total is £500.
Please re-extract, ensuring all line items are captured.
```

Without the specific error the model has no guidance and **usually reproduces the same mistake**. With it, self-correction is targeted: re-scan for missed line items, check field placement, recompute.

### The retry effectiveness boundary — the most aggressively tested idea here

| Retries **are** effective for | Retries are **not** effective for |
|---|---|
| Format mismatches (wrong date format, currency notation) | Information **genuinely absent** from the source |
| Structural errors (wrong field, incorrect nesting) | Data that exists only in an **external document not provided** |
| Misplaced values (present in the document, extracted to the wrong field) | Fields requiring knowledge the model doesn't have |
| **Mathematical errors** (a missed line item throwing off the total) | |

**If the document genuinely lacks a value, no amount of retrying produces a correct one.** Flag for human review, or return `null` if the schema allows it.

> The exam presents **both** kinds in one question — e.g. Document A whose line items don't sum (retry with the discrepancy error) and Document B missing a field entirely (flag for review). Answering "retry both" or "review both" are the two wrong extremes.

### Build self-correction into the schema

Rather than relying only on external validation logic, let the extraction surface its own problems:

| Pattern | Shape | What it catches |
|---|---|---|
| **`calculated_total` vs `stated_total`** | Both extracted; plus a `total_discrepancy` boolean | Sum mismatches, with no external logic needed |
| **`conflict_detected`** | Boolean set when the source contains contradictory information | A document saying "payment due: 30 days" in one section and "payment terms: net 60" in another — extract **both** and flag, rather than silently picking one |
| **`detected_pattern`** | A string on each finding naming the construct that triggered it | Enables dismissal analysis: if reviewers consistently dismiss findings triggered by one pattern, that pattern needs prompt refinement |

`detected_pattern` creates a **systematic improvement loop**: extract → validate → collect dismissal data → refine prompts → repeat.

### Pydantic as the validation layer

The exam guide names Pydantic alongside JSON Schema. A Pydantic model does **two jobs at once**:

| Job | Enforces |
|---|---|
| **Parsing** | Structure — types, required fields, enums |
| **Validators** | Semantics — the rules a JSON schema **cannot** express: cross-field arithmetic, date ordering |

Both surface through one `ValidationError` carrying machine-readable, per-field messages — which is exactly the third ingredient the retry loop needs, in a form you can format straight into the prompt.

> **Trap: treating Pydantic as redundant once `tool_use` enforces a schema.** Schemas eliminate syntax errors but cannot express cross-field rules. The platform enforces the schema; your validators enforce the business rules; the retry loop consumes whichever fails.

The retry-effectiveness boundary applies unchanged: **a validator failing because information is absent still means human review, not a retry.**

### Retry with the specific error, not the same request

> Extraction fails Pydantic validation with a specific error: `expected float for quantity, got '2 to 3'`. Retrying the identical request produces an identical failure.

**Fix: send a follow-up request containing the validation error and the failed output, asking the model to correct that specific defect.**

An identical request to a (near-)deterministic function yields an identical result. The retry only helps if the input changed — and the new input is **the error message**.

```
Your previous extraction failed validation:
  quantity: expected float, got '2 to 3'

Here is what you produced: {...}
Please correct the quantity field. Ranges should be recorded as the midpoint.
```

- *Set temperature to 0* — makes the failure perfectly reproducible. Backwards.
- *Pre-process the source documents* — upstream work to avoid teaching the model a conversion it can do.
- *Upgrade the model tier* — doesn't communicate the schema requirement that was violated.

### Semantic errors need the document too

> Invoice extraction via tool use produces syntactically valid JSON, but **12%** fail *semantic* validation — line item sums don't match totals, vendor IDs don't match the expected format. These currently route to manual review.

**Fix: on validation failure, make a follow-up request containing the source document, the extraction, and the specific validation errors.**

Note what's different from the syntax case: **the source document must be included.** A format error can be fixed from the output alone; a semantic error requires re-reading the source to find the right value.

- *Blind retries* — same reasoning, same invalid result.
- *Post-processing logic that auto-corrects the amounts* — silently modifies values (see 4.3).
- *Stricter schema constraints* — a schema can enforce that a value is a float; it cannot enforce that the floats sum correctly. **Schema guards shape; semantics need checks.**

### When retrying cannot possibly work

> A system retries with error feedback, appending validation errors each time; most failures resolve within 2–3 attempts. **Which failure pattern would additional retries be LEAST effective against?**

**Answer: when the required information is absent from the input entirely** — e.g. the model extracts `"et al."` for co-authors because the full author list exists only in an external document that was never provided.

**You cannot retry your way to data the model was never given.** No amount of error feedback conjures missing information. The fix is upstream: supply the missing source.

The three retryable patterns, for contrast:

| Pattern | Retryable? | Why |
|---|---|---|
| ISO datetime supplied where `YYYY-MM-DD` is wanted | ✅ | The correct date is derivable from what was provided |
| Locale-formatted number `"1,234"` instead of `1234` | ✅ | Pure formatting transformation |
| Nested object where a flat array is required | ✅ | Pure structural transformation |
| Co-authors listed only in an unprovided document | ❌ | **The information does not exist in the input** |

**Diagnostic: can the correct value be derived from what the model was given? If yes, retry with feedback. If no, fix the input.**

### Detecting the semantic error in the first place

Retry-with-feedback presupposes that something *noticed* the error. For cross-field arithmetic, the cheapest detector lives in the schema itself.

> An invoice extraction pipeline extracts line items, subtotals, tax, and grand totals. In **18%** of extractions, the line items don't sum to the extracted grand total — a mix of OCR errors in the source and model errors in the extraction. Downstream accounting systems reject mismatched records.

**Fix: extract a `calculated_total` (the model's sum of the line items) alongside the `stated_total` (the figure printed on the invoice), and flag records for human review when they differ.**

Two properties make this the right answer:

1. **It costs nothing extra.** Both values are already present in the document; you are asking for one more field, not one more model.
2. **It surfaces rather than resolves.** You cannot tell from the extraction alone which number is wrong — the source itself may be inconsistent. Raising a flag is the correct output.

- *A separate validation model to reconcile the discrepancy* — cost and complexity to guess at something undeterminable from the available information.
- *Few-shot examples of consistent extractions* — may reduce model-side errors; cannot fix an OCR error in the source, and flags nothing.
- *Post-process to adjust the line items proportionally* — **the worst option.** It manufactures agreement, destroys the only signal that something was wrong, and writes fabricated figures into an accounting system.

**Principle: a cross-field consistency check raises a flag; it never performs a silent correction.** This question was previously filed under 3.6 because the scenario is set in a pipeline — but the mechanism is validation, which is why it belongs here. The same design also appears in [4.3](#43-structured-output-with-tool-use) as a schema pattern; it is one idea wearing two labels.

### The validation ladder

```
1. Syntax / shape      → tool use with a schema          (prevents)
2. Semantic / business → cross-field + business-rule checks (detects)
3. Correction          → retry with the specific error + source (repairs)
4. Residual risk       → confidence-based routing to human review (contains)
```

### Exam traps for 4.4

- **Assuming retries always work.** The exam pairs a fixable and an unfixable document in one question.
- **Retrying without including the specific validation error** — naive retries reproduce the same mistake.
- Relying on schema validation alone, with no semantic checks.
- **Treating Pydantic as redundant** once `tool_use` enforces a schema.
- Blind retry, or "retry everything" as a blanket policy.
- Temperature or model-tier changes for a validation failure.
- Trying to recover information that was never in the input.
- Auto-correcting instead of surfacing.

---

## 4.5 Batch processing strategies

The whole subtopic rests on one trade-off.

### The Message Batches API: the fixed constraints

| Constraint | Detail |
|---|---|
| **Cost** | **50% savings** vs. synchronous |
| **Processing window** | **Up to 24 hours** — may arrive in minutes, may take the full window |
| **Latency SLA** | **None.** You cannot rely on results arriving in any specific timeframe |
| **Tool calling** | **No multi-turn tool calling within a single batch request** |
| **Correlation** | **`custom_id`** on each request, used to match it with its response |

### The matching rule — the single most tested concept here

| Use | For |
|---|---|
| **Synchronous API** | **Blocking** workflows — someone or something is waiting. Pre-merge checks, real-time review feedback, anything where developers are blocked |
| **Batch API** | **Latency-tolerant** workflows — results consumed later. Overnight technical-debt reports, weekly audits, nightly test generation, bulk extraction |

> **Sample Question 11:** a manager proposes moving *everything* to batch for the cost savings. **The correct answer keeps blocking workflows synchronous and moves only the latency-tolerant ones.** Variants offering a timeout fallback to real-time, or status polling, are still batch for a blocking workflow.

**Trap: assuming batch results arrive quickly because they usually do.** There is no SLA. **Design around the 24-hour maximum, not the best case.**

### SLA arithmetic

The exam may ask you to work backwards from an SLA to a submission frequency.

> Requirement: a **30-hour** SLA.

```
30h SLA - 24h worst-case processing = 6h of buffer
  -> buffer absorbs request collection, input validation, operational delays
  -> submit every 4h so a fresh batch is always in flight
  -> a 6h cadence leaves no margin at all
```

The 24 hours is a **maximum window, not a delivery guarantee**. A batch that doesn't finish inside it returns **`expired`** — size the schedule against that worst case and treat an expired batch as a resubmission.

### The multi-turn tool calling limitation

The batch API does **not** support multi-turn tool calling within a single request. You cannot:

- Define tools and have the model call them mid-request
- Process tool results and continue within the same batch item
- Run an agentic loop inside one batch request

> **Direct exam test point:** if a scenario describes a batch workflow that needs to execute tools **during** processing, the answer is to use the **synchronous API** for that step.

*(Current docs split this by tool type — server tools run the same agentic loop inside a batch and can return a paused stop reason, while client tools your own code executes still cannot complete a loop in-batch. **Answer per the guide: no multi-turn tool calling in a batch request.**)*

### Refine the prompt on a sample set first

The most cost-effective strategy is proactive:

1. **Sample set testing** — take **5–10 representative documents** covering the range of formats and edge cases.
2. **Iterate on the sample** — refine prompts, add examples, adjust the schema until accuracy is high.
3. **Submit the full batch** with the refined prompt.
4. **Handle failures** — resubmit only failures, with targeted modifications.

The arithmetic: on 1,000 documents, a **90%** first-pass rate means 100 retries; a **60%** rate means 400 — **four times the resubmission cost**, plus the batch cost for those retries.

### Failure handling — three steps

1. **Identify failures by `custom_id`.**
2. **Resubmit only the failures, with modifications** — chunk oversized documents, simplify prompts for unusual structures, add format-specific examples. **Never resubmit the whole batch.**
3. **Treat `expired` as a failure too** — that's what an overrun batch returns.

| | Messages API (real-time) | **Message Batches API** |
|---|---|---|
| Cost | Standard | **~50% discount** |
| Latency | Seconds | **Up to 24 hours** |
| Use for | Blocking checks, SLAs under ~24h | Anything latency-tolerant |
| Identifies results by | The response | **`custom_id`** per request |

**The decision procedure: compute the slack. If the deadline comfortably exceeds the 24-hour batch window, batch it. If not, use real-time.**

### Does the SLA leave room?

> Documents arrive continuously through business hours. Extraction results must be available within **30 hours** of arrival at 99.9% reliability. The Batches API gives a 50% discount with an up-to-24-hour window.

**Fix: submit a single end-of-day batch containing that day's documents.** 24 hours of processing plus buffer fits inside 30 hours, and one daily batch maximizes efficiency.

- *Submit every 4–6 hours* — operational overhead for no benefit; the 24-hour window already fits.
- *Use the real-time API for everything* — forfeits the 50% discount with no SLA justification.

> A CI/CD pipeline generates release notes by classifying and summarizing **~200 commits weekly**, currently one Messages API call per commit on Sonnet. Notes are needed within **~12 hours**. Reduce per-token cost while keeping the same model, prompts, and output quality.

**Fix: submit all 200 as one Batches API job with unique `custom_id` values and collect the results when it finishes.** 12 hours comfortably covers the window; the 50% discount applies to all tokens; model and prompts are unchanged.

- *Fire the 200 Messages API requests concurrently* — faster wall-clock, identical per-token price. **Parallelism is not a discount.**
- *Concatenate all 200 commits into one large request* — still the Messages API, still no batch discount, plus quality risk from cramming 200 commits into one context.
- *Switch to Haiku* — cheaper, but violates the explicit "retain the same model" requirement.

### Split by latency requirement

> A pipeline handles two document types with the same JSON schema: standard monthly reports (archived after processing) and urgent exception reports (**must trigger alerts within 30 minutes**). Minimize cost while meeting latency.

**Fix: route standard reports to the Batches API and urgent exceptions to the real-time Messages API.**

- *Send everything to the Batches API and process urgent ones "immediately" when results arrive* — the batch window can run up to 24 hours. A 30-minute SLA cannot survive that.
- *Send everything real-time* — meets the SLA, forfeits 50% on documents that didn't need speed.
- *Hourly batches with an expedited flag* — hourly *submission* does not change *processing* latency, and there is no expedite mechanism.

**When a scenario presents two populations with different urgency, split them.**

### Resubmit only the failures

> A daily batch of **10,000** documents completes. **300 (3%)** fail with `context_length_exceeded`; the results identify failures by `custom_id`. What's the most cost-effective recovery?

**Fix: chunk the 300 failed documents into smaller pieces, resubmit only those, and combine the partial extractions.**

Two requirements in one answer: address the *cause* (input too long → chunk it) and avoid repeating 9,700 successful jobs.

- *Reprocess the entire batch with prompt caching* — pays for 9,700 successes again.
- *Increase `max_tokens`* — **a precise misunderstanding.** `max_tokens` caps the *output*. The error is about *input* length. This cannot help.
- *Switch to a larger-context model for the whole batch* — might fix the 300, wastes money on the 9,700.

### Iterate on a large corpus

> **50,000** legacy contracts, **2-week** deadline. Sample testing shows **82%** pass on the first attempt; the **18%** that fail have diverse issues — missing fields, malformed dates, party identification problems — needing targeted prompt refinement.

**Fix: submit all 50,000 via the Batches API, analyze the failures, then resubmit just the failures in successive batches with progressively refined prompts.**

You get the full, real failure population to analyze, discounted pricing throughout, and convergence well inside two weeks.

- *Sequential batches of 5,000* — you analyze a smaller slice at a time and risk missing patterns that only appear across the whole corpus.
- *Real-time for everything* — forfeits the discount on 50,000 documents.
- *Sample 2,000 via real-time first to tune the prompt* — spends time and premium pricing on a proxy population when running the real 50,000 gives you both the results and the complete failure set.

### Chunking for attention, not just for limits

> Meeting transcript extraction achieves **94%** accuracy on short transcripts (<30 min) but only **68%** on long ones (>60 min) with scattered key information. **Both fit within the context window.**

That last clause is the key: this is **not** a context-limit problem. It's an attention-dilution problem — evidence scattered across a long document competes for focus.

**Fix: split long transcripts into chunks, extract from each separately, then merge and deduplicate.**

- *Few-shot examples drawn from lengthy meetings* — shows the desired output; doesn't concentrate attention.
- *A larger model tier* — may help marginally; doesn't address the scatter.
- *Generate a summary first, then extract from the summary* — **actively harmful.** Summarization discards details before extraction, and those details are exactly what you're extracting.

**Two reasons to chunk:** the input exceeds the limit (the 300-failure case), or attention is spread too thin across a long input (this case). Both are solved by chunk → extract → merge.

### Exam traps for 4.5

- **Switching all workflows to batch for the cost savings** — blocking workflows must stay synchronous.
- **Assuming batch results arrive quickly because they usually do.** There is no SLA; design around 24 hours.
- **Using batch for a workflow that needs multi-turn tool calling** mid-processing.
- Using real-time when the deadline comfortably exceeds 24 hours.
- Treating concurrency as a cost reduction.
- Reprocessing an entire batch to fix a small failure subset.
- Forgetting that `expired` is a failure state requiring resubmission.
- Raising `max_tokens` for an input-length error.
- Summarizing before extracting.

---

## 4.6 Multi-instance and multi-pass review

Six questions, unified by one idea: **when a single pass is asked to do two things that pull against each other, split it into two passes.**

### The multi-pass review architecture

For large reviews the guide prescribes a specific two-pass shape:

| Pass | Scope | Catches |
|---|---|---|
| **1. Per-file local analysis** | Each file reviewed **individually** with a focused prompt | Local bugs, security issues, logic errors — at **consistent depth**, because each invocation sees one file |
| **2. Cross-file integration** | A separate pass receiving **all** per-file findings | Data flow between modules, API contract violations, dependency conflicts — **and contradictions among the per-file findings themselves** |

That last item is easy to miss: the integration pass also reconciles the per-file passes against each other, which is what stops the same pattern being flagged in one file and approved in another.

### Why a larger context window does not fix this

> **The exam includes this as a specific distractor:** "switch to a higher-tier model with a larger context window."

It sounds reasonable — if the model can't handle 14 files, give it more capacity. **But the problem isn't context size, it's attention *quality*.** A bigger window doesn't stop the model spreading attention unevenly. Only focused per-file passes guarantee consistent depth.

Two more distractors from the same question:

- *Run three independent passes over the whole PR and flag only what 2 of 3 agree on* — triples cost, and consensus among three equally diluted passes is not depth.
- *Require developers to split large PRs into 3–4 files* — pushes the architectural problem onto humans and changes their workflow.

### Confidence-based routing — and calibration

The model can self-report confidence per finding, enabling a routing strategy:

| Confidence | Route |
|---|---|
| High | Report directly |
| Low | Human review |

```json
{"finding": "Potential race condition in order processing",
 "severity": "major", "confidence": 0.65,
 "reasoning": "Lock acquisition looks correct but unlock timing depends on an
               async callback whose ordering I cannot fully verify.",
 "route": "human_review"}
```

**The confidence score is not self-reported accuracy** — it is the model's read on its own certainty, and it is **poorly calibrated out of the box**.

> **The distinction the exam draws:** *raw* confidence scores (uncalibrated, unreliable for automated decisions) vs. *calibrated* thresholds (validated against **labelled validation sets**, suitable for routing). **Using uncalibrated confidence for automated decisions is an anti-pattern.**

Calibrate by running examples whose answers you already know, measuring how reported confidence tracks actual accuracy, then setting thresholds from that data.

### The full production architecture

1. **Generation** — first instance produces the code/extraction/analysis.
2. **Per-unit review** — independent instances review each unit individually.
3. **Integration review** — a separate instance checks cross-unit consistency.
4. **Confidence routing** — low-confidence findings go to human review.
5. **Calibration loop** — labelled validation sets continuously recalibrate the thresholds.

This is more expensive than single-pass review. **It is worth it when review quality directly affects production reliability** — CI/CD gates, financial extraction, compliance analysis, anywhere a missed issue has downstream consequences.

### Conflicting constraints: finding vs. filtering

> An automated PR review prompt says *"only report high-confidence issues you are certain about"* and *"err on the side of not commenting."* Result: high precision, low recall. A race condition that caused a production outage **was visible in the reviewed PR** and went unreported.

**Fix: split the review into two passes.**

1. **Finding pass** — goal is *coverage*. Flag every potential issue, each annotated with confidence and severity.
2. **Thresholding pass** — goal is *precision*. Decide what's worth reporting.

The single prompt asked the model to maximize coverage and suppress noise simultaneously. Those objectives are in direct tension, and the conservative instruction won — suppressing a real bug. Separating them lets each pass optimize one objective, and makes the precision/recall trade-off an explicit, tunable threshold instead of an emergent property of prompt wording.

- *Add more context — git history, tests, dependency graph* — richer input, same conflicting instructions.
- *Remove the conservative filtering and apply a programmatic post-filter* — closer, but a keyword/heuristic filter can't judge severity the way a reasoning pass can, and it's still one generation step.
- *Add few-shot examples while keeping the conservative filtering* — the tension remains.

### Competing concerns dilute each other

> A single review prompt covers security, API design, and business logic correctness. Recall: **API design 82%** (strong), **business logic 34%** (poor). Adding logic-bug examples raises logic recall to **41%** but drops API design recall to **68%**.

That trade-off is the diagnostic. **One prompt has a fixed attention budget; strengthening one concern weakens another.**

**Fix: split into separate focused prompts — one for security and API design, another for business logic — each with its own dedicated examples, then consolidate the findings before posting.**

- *Add a detailed checklist of logic edge cases* — more prose competing in the same prompt.
- *Upgrade the model tier* — doesn't resolve competing instructions; the dilution is structural.
- *Provide the full repository as context* — more input, same conflict, worse signal-to-noise.

**Rule: when improving one concern measurably degrades another, the prompt is overloaded. Split it.**

### Output truncation

> An automated review uses `tool_use` with a `report_findings` tool returning a JSON array of findings (`file_path`, `line_number`, `severity`, `category`, `description`). A large PR touching **30+ files** hits the `max_tokens` limit; the output truncates mid-JSON and the parser fails.

**Fix: split the review into multiple API calls, each analyzing a subset of the changed files, then merge the findings arrays.**

Each response stays below the output ceiling, and the structured contract is preserved.

- *Increase `max_tokens` and shorten the descriptions* — buys headroom; the next larger PR truncates again. Treats a symptom.
- *Switch to a markdown list instead of structured output* — abandons the structured contract that everything downstream depends on, to solve a size problem.
- *Retry with a filter for critical/high severity only* — reduces volume without guaranteeing anything, and silently drops findings.

**Note the family resemblance to 4.5: input too long → chunk the input; output too long → chunk the work.**

### Independence: a fresh instance reviews better

> A developer uses Claude Code to refactor a function, then asks **the same session** to review the code for issues. A separate CI review later catches several bugs the same-session review missed.

**Cause: the session retains its own prior reasoning and assumptions, making it less likely to question code it just wrote.** A fresh CI review has no such attachment and examines the diff without anchoring bias.

The distractors are plausible-but-secondary:

- *CI has access to the full codebase while the local session only saw the current file* — generally false; a local session has the same repository access.
- *The CI review uses a more specific prompt* — possible, and it might contribute, but **independence is the primary mechanism.** The question is testing whether you identify the root cause or a contributing factor.
- *The session's extended context was exhausted* — a real phenomenon (4.1), but not what's driving this specific miss.

**Rule: separate the generating context from the reviewing context whenever self-justification could bias the reviewer.** This is the D4 echo of Domain 3's "review in a separate invocation."

### When to use multiple passes

| Situation | Pattern |
|---|---|
| Recall is low because the prompt also demands precision | Finding pass + thresholding pass |
| Improving one concern degrades another | Separate focused prompts per concern, then merge |
| Output exceeds `max_tokens` | Split the input across calls, merge the results |
| The reviewer wrote the code | A fresh, independent instance |
| High-stakes fields needing certainty | Independent passes, reconcile disagreements |

When merging findings from multiple passes, deduplicate and apply consistent severity rules — otherwise the same issue appears several times with different labels.

### Exam traps for 4.6

- **Choosing self-review in the same session** as a viable review strategy.
- Using a single pass for a large multi-file review.
- **Switching to a larger context window to fix attention dilution** — it's attention quality, not capacity.
- **Using uncalibrated confidence scores for automated routing.**
- Keeping one prompt and adding examples when the scenario shows a precision/recall trade-off.
- Raising `max_tokens` as a truncation fix.
- Abandoning structured output to dodge a size limit.
- Running N independent full passes and voting, instead of focusing each pass.

---

## The seven sentences that answer most of Domain 4

1. **Schema-valid ≠ correct.** Shape is guaranteed; meaning is not.
2. **Required fields cause fabrication.** Give the model a legal way to say "not stated" (optional/null, empty array) and "ambiguous" (`"unclear"`).
3. **Confidence fields never prevent fabrication.** The model just rates its own guess.
4. **Examples teach judgment; instructions direct process; schemas constrain shape.** Match the tool to the gap.
5. **Retry with the specific error — and only when the answer is derivable from the input.** Missing information is not retryable.
6. **Batch when the deadline comfortably exceeds 24 hours; split populations by urgency.** Parallelism is not a discount.
7. **Split a pass that is being asked to do two opposing things**, and never let the author review its own work.

## Study order within this domain

1. **4.3 structured output** — 17 questions (~9 concepts); memorize the schema design checklist, especially optional-vs-required and the enum escape hatch.
2. **4.1 system prompts** — 10 questions; drift diagnosis, versioning, and the assume-state-invite ambiguity pattern.
3. **4.2 few-shot** — 10 questions (~6 concepts); learn the four distractor families and you can answer these mechanically.
4. **4.5 batch processing** — 8 questions; pure arithmetic on latency slack.
5. **4.6 multi-pass** — 6 questions; the conflicting-constraints diagnosis.
6. **4.4 validation & retry** — 6 questions; the derivable-vs-missing distinction and the consistency-check design.
7. **[4.0 cross-topic questions](../../categorized_exam_dump/domain-4-prompt-engineering-structured-output/4.0-cross-topic-and-domain-level.md)** — eight authored scenarios drilling the shape/judgment/meaning triage without naming which layer is broken.

## Links

[Domain 4](https://claudecertificationguide.com/learn/4-prompt-engineering) ·
[4.1 System prompts](https://claudecertificationguide.com/learn/4-prompt-engineering/4-1-system-prompts) ·
[4.2 Few-shot prompting](https://claudecertificationguide.com/learn/4-prompt-engineering/4-2-few-shot-prompting) ·
[4.3 Structured output](https://claudecertificationguide.com/learn/4-prompt-engineering/4-3-structured-output) ·
[4.4 Validation & retry loops](https://claudecertificationguide.com/learn/4-prompt-engineering/4-4-validation-retry-loops) ·
[4.5 Batch processing](https://claudecertificationguide.com/learn/4-prompt-engineering/4-5-batch-processing) ·
[4.6 Multi-pass review](https://claudecertificationguide.com/learn/4-prompt-engineering/4-6-multi-pass-review)
