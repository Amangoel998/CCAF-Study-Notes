# Pattern Decoder — Generalizable Decision Rules

**What this file is:** a set of **decision axes**. Each one is a property of the *scenario structure* that determines the answer by definition, independent of the domain, the vocabulary, or the numbers used to dress it up.

**What this file deliberately is not:** a list of phrases to match. Memorizing that "diverse and evolving" means *dynamic routing* fails the moment the exam writes "the mix of request types shifts each quarter." The practice bank was used only to **verify** that these axes hold — never as the source of the trigger.

**Test for whether something belongs here:** could you derive the rule from first principles, without having seen a single practice question? If no, it's an artifact of one bank and it's excluded.

---

## How to read any scenario

Before looking at the options, answer four questions:

1. **What is the required strength of the rule?** Absolute, or preferential? → Part 1
2. **What exactly is broken?** Is the data absent, degraded, malformed, or merely wrong? → Part 2
3. **What has the scenario explicitly fixed or ruled out?** → Part 3
4. **What structural property of the problem are they probing?** Variability, scope, reversibility, comparability? → Part 4

Three of the four options will usually fail one of these before you evaluate it on merit.

---

## Part 1 — Requirement strength determines the mechanism

This is the single most reusable axis on the exam. Rank every available mechanism by **how hard it is for the model to route around**, then match it to how strongly the rule must hold.

```
HARD ─────────────────────────────────────────────────► SOFT
server-side    permission     deterministic    orchestration    prompt-level
logic inside   denial         hook on the      check around     guidance
the tool                      tool call        the loop
```

| Required strength of the rule | Correct mechanism class |
|---|---|
| **Must never happen** (prohibition, authorization boundary, spend limit) | Server-side logic in the tool, or a permission denial |
| **Must always happen, mechanically** (a transformation, a check with no judgment) | A deterministic hook on the relevant tool event |
| **Must be true when the process ends**, regardless of how it ended | A check in the layer that **outlives** the process |
| **Should generally happen**, judgment permitted | Prompt-level guidance (system prompt, project config) |
| **Should happen when working on a particular kind of thing** | Conditionally loaded guidance scoped by pattern |

### The derived rules

| Observation in the scenario | Necessary conclusion | Why it's definitional |
|---|---|---|
| A rule is stated as absolute **and** the current mechanism is prompt-level **and** any non-zero violation rate is reported | The **mechanism class** is wrong, not its wording | A probabilistic process cannot satisfy a binary requirement. Strengthening the language changes the probability, not the class |
| Emphasis or examples **reduced** the violation rate without eliminating it | Same conclusion, stated more explicitly | A reduction proves the lever operates on probability. More of a probabilistic lever is still probabilistic |
| The control point is **the model choosing** between a safe and an unsafe option | That choice is the vulnerability | Any control that depends on correct selection fails exactly when selection fails |
| The control is a **parameter or flag the model sets** (a dry-run boolean, an "approved" field) | Bypassable | You have asked the component you don't trust to certify itself |
| The guarantee concerns **how a process terminates** | It must be implemented outside that process | Anything inside the process can be skipped by the process ending early |
| The rule is intrinsic to **what the tool itself does** | Enforce inside the tool | The tool is the last point before the effect; nothing downstream of it can be bypassed |
| The rule is about **intercepting and redirecting** a call to a tool you don't control | Enforce at the hook / interception layer | You cannot modify internals you don't own |

### The inverse, which is equally tested

| Observation | Conclusion |
|---|---|
| The rule is a **style or preference**, and occasional deviation is tolerable | Leave it at prompt level. Hard enforcement here is over-engineering |
| A scenario lists several requirements of **differing strength** | Allocate each to a different mechanism. An option applying one mechanism uniformly is wrong |

---

## Part 2 — Symptom shape determines root cause

Distinct failure modes produce distinct observable signatures. The exam routinely offers all of them as options for one symptom; the signature disambiguates.

| Observable signature | Root cause | Fix class |
|---|---|---|
| Downstream step reports it received **nothing** | A hand-off was never made | Put the data into the downstream input explicitly |
| Output is **present but partial** — early content dropped, detail thinned | Capacity overflow | Reduce what's sent; compress the compressible part |
| Output is **present but imprecise** — specifics replaced by generalities, earlier commitments softened | Attention dilution: accumulated content competing with instructions | Re-assert, relocate, or isolate the content that must dominate |
| Output is **structurally broken** — cut off mid-structure, unparseable | Output ceiling reached | Split the work so each response stays under the ceiling |
| Output is **well-formed but wrong** | Shape was enforced; meaning was not | Semantic checks, not stricter structure |
| An operation **succeeds, then a later identical operation fails** | State changed between the two calls | Make the sequence atomic |
| The component **narrates** doing something it never did, with no error | The capability isn't available to it | Grant or expose the capability |

### The non-negotiable distinction

> **Total absence is not degradation.** "It received nothing" and "it received a weaker version" have different causes and different fixes.

Capacity explanations predict **partial or thinned** output. They cannot produce **complete absence**. Whenever an option attributes a total absence to capacity, it is wrong by construction — and the reverse is equally true.

### Onset and distribution

| Pattern over time | What it must be | What it cannot be |
|---|---|---|
| Fails on the **first** attempt | Something structurally missing | Accumulation, drift, exhaustion — none of which have had time to occur |
| Correct early, **gradually** degrades | Accumulation or dilution | "The instruction is only sent once" — that would fail abruptly at the second step, not gradually |
| Fails **abruptly** at a threshold | A limit was crossed | Gradual dilution |
| Fails **intermittently** on identical input | Non-determinism, or an external dependency | A deterministic configuration error |

### Transport, capacity, and content are independent

| Claimed cause | Why it's almost always wrong |
|---|---|
| "They need a shared connection / session / channel" | Transport does not create shared state. Two components over one connection still hold independent state |
| "The window is too small" (when nothing arrived) | See above |
| "It needs an instruction to remember" | Recall comes from what is *sent*, not from being told to recall. No instruction recovers unsent data |

---

## Part 3 — Explicit qualifiers eliminate option classes

When a scenario states a fact that is not needed to understand the situation, it is there to eliminate options. Read every such clause as a constraint.

| Class of qualifier | What it eliminates |
|---|---|
| A resource is stated to be **far from its limit** | Every explanation and fix based on that resource's capacity |
| A failure mode is stated to **persist after** an attempted fix | That entire fix class, including stronger versions of it |
| An attempted fix is stated to have **partially worked** | That fix class — partial success on a binary requirement is failure |
| A cost is **forbidden** (added latency, added complexity, extra passes, extra models) | Every option incurring it, regardless of merit |
| A component is **required to stay fixed** (same model, same prompts, same output quality) | Every option that changes it |
| An enforcement **location** is specified | Every option enforcing elsewhere, even if functionally similar |
| Two sub-populations are described with **different requirements** | Any uniform treatment of both |
| The task is described as **one-off** | Permanent configuration |
| The task is described as **recurring and shared** | Ad-hoc, per-invocation, or per-machine solutions |
| Prior work is stated to be **still valid** | Starting over |
| Prior information is stated to be **now incorrect** | Retaining it and instructing around it |
| Something is stated to **keep changing** | Any fixed enumeration of it |

### The composite rule

Where a scenario lists **several distinct symptoms**, the correct option addresses **all** of them. The exam reliably includes an option that resolves most but not all — it is the most attractive wrong answer in the set. Count the symptoms, then count what each option covers.

---

## Part 4 — Structural properties that decide the design

These are the axes the exam probes once you've got past diagnosis. Each is a property of the problem, not of the wording.

### 4a. Does the work vary between runs?

| Property | Decomposition pattern |
|---|---|
| The same steps execute every time | Sequential chaining. An orchestrator that *decides* which subtasks to run has nothing to decide |
| The next step depends on what the previous step found | Adaptive subtask generation |
| Scope is open-ended and must be triaged | Survey cheaply → rank by impact → keep the plan revisable |
| Subtasks are independent and known | Dispatch them concurrently |

Corollary: **parallel dispatch is right when subtasks are independent and wrong when each must inform the next.** Independence, not count, is the criterion.

### 4b. Is the set open or closed?

| Property | Design |
|---|---|
| The set of valid values is closed and stable | Constrain it |
| The set is open, or new members keep appearing | Constrain the known members, add an explicit escape member, and capture the raw value alongside it |
| The set of inputs to classify keeps shifting | Let the model judge; any fixed classifier or category list goes stale by construction |

Reactively extending an enumeration is always wrong when the scenario establishes that the set grows.

### 4c. Can the representation express the true state of the world?

This is the root of most structured-output questions.

| True state that must be expressible | Required representation |
|---|---|
| The information is **not present** in the source | A legal "absent" value — optional, nullable, or empty collection |
| The information is present but **genuinely indeterminate** | An explicit "indeterminate" member, distinct from any substantive value |
| The fact has **multiple values over time** | A collection of values, each carrying its own source and effective date |
| The fact is **contested between sources** | Separate presentation of each, with its own characterization |

**The derived rule:** a mandatory field is an instruction to produce a value. If the world may not supply one, the model will. Making fields mandatory to improve reliability inverts the outcome.

**The corollary:** a self-reported certainty score never prevents invention — it annotates it. Certainty scores are for **routing decisions**, never for suppressing fabrication.

### 4d. Does validation enforce format or identity?

| What's ambiguous | What actually fixes it |
|---|---|
| The **format** of a value | Constraints: patterns, types, enumerations |
| **Which entity** is referred to | An identifier obtained from a lookup — constraints cannot disambiguate two correctly-formatted candidates |

Any option that tightens format when the scenario includes a *which-one* failure resolves only part of the problem. (See the composite rule in Part 3.)

### 4e. Is the failure in the call or in the outcome?

| Property | Classification |
|---|---|
| The request is **malformed** — required input missing, contract violated | Protocol-level error. It could not have been executed |
| The request was **well-formed and executed**, and the result was a failure | A normal result carrying an error flag |

The downstream system returning "not found" or "unavailable" is an **outcome**, however severe it sounds. The test is whether the call itself was valid, never how bad the failure feels.

### 4f. Who should handle a failure?

| Property of the failure | Handler |
|---|---|
| **Transient** — the same call may succeed later | The tool, with bounded retry. Surfacing it only wastes the model's turns |
| **Permanent or business-rule** — the same call will never succeed | Surface it to the model with a category and a next step. Retrying is pure waste |
| Either | The payload must carry enough structure for the recipient to choose. Undifferentiated errors produce inconsistent recovery, which is itself a recognizable signature |

**Retryability test:** is the correct value **derivable** from what the component was given?

| Derivable? | Response |
|---|---|
| Yes — it's a format, structure, or transformation problem | Retry, feeding back the specific error. An unchanged retry of a near-deterministic process returns the same result |
| No — the information was never in the input | No amount of retrying or feedback can conjure it. Fix the input |

### 4g. Reversibility and materiality

| Property of the pending action | Behaviour |
|---|---|
| Reversible, and the unknowns are **parameters** of one action | Infer from context, **state the inference explicitly**, act, invite correction |
| Irreversible, or the unknowns determine **which action entirely** | Ask one targeted question |
| Either | Never a battery of questions; never a silent default |

The explicit statement of the assumption is load-bearing. A silent default and a stated default are different answers, and only one preserves the user's ability to correct it.

### 4h. Scope follows audience and frequency

| Property | Where it belongs |
|---|---|
| Shared by a team, must stay synchronized | Committed to the shared repository |
| Personal, experimental, or machine-specific | User-level configuration |
| Relevant to **every** interaction | Always-loaded guidance |
| Relevant only when working on a **particular kind of file** | Pattern-scoped conditional loading |
| Relevant only when performing a **particular task** | On-demand invocation |
| Needed exactly once | The prompt itself |

Two consequences the exam leans on: user-level configuration never reaches a collaborator, and always-loaded content costs context in every unrelated session.

### 4i. Competing objectives in one pass

| Property | Design |
|---|---|
| One step is asked to maximize coverage **and** minimize noise | Split: a discovery step optimizing coverage, then a separate thresholding step |
| Strengthening one concern **measurably weakens** another | The step is overloaded. Split by concern and merge the results |
| The evaluator also produced the thing being evaluated | Use an independent instance. Shared context anchors on its own prior reasoning |
| Two options must be compared fairly from shared background | Branch from the shared point. Sequential exploration in one context lets the first bias the second |

The measured trade-off is the diagnostic: if improving A degrades B, no amount of added instruction, context, or capability resolves it — only separation does.

### 4j. Aggregate versus segment

| Property | Rule |
|---|---|
| A quality metric is reported **in aggregate** before a decision to reduce oversight | Disaggregate first. An aggregate is a weighted average and can conceal a segment below the acceptable floor |
| A routing rule sends only part of the output for inspection | The un-inspected portion has **no measurement**. Sample it continuously, or you are blind by design |
| Detection relies on repeating the same process and comparing | Correlated errors survive. Agreement is not accuracy |

### 4k. Latency slack

| Property | Choice |
|---|---|
| The deadline comfortably exceeds the deferred-processing window | Use the deferred, discounted path |
| The deadline is shorter than that window | Use the immediate path |
| Two populations have different deadlines | Route separately. A uniform choice either overpays or misses an obligation |

Related: **concurrency reduces elapsed time, never unit price.** An option offering parallelism as a cost reduction is wrong by definition.

### 4l. Which limit was exceeded

| Property | Fix |
|---|---|
| The **input** exceeded a limit | Reduce or divide the input |
| The **output** exceeded a limit | Divide the work so each response is smaller |
| **Neither** limit was reached, but quality falls as input grows | Attention dilution — divide anyway, to concentrate each pass |

Raising an output ceiling never addresses an input-length failure, and raising either never addresses dilution.

### 4m. Where information is lost in a pipeline

| Property | Rule |
|---|---|
| A stage **compresses** its input | That stage is where metadata disappears unless carried in a separate channel |
| Attribution, timing, or methodology is needed downstream | Carry it as structure through every stage. Reconstructing it afterwards by similarity misattributes |
| Observations differ and carry **different collection times** | Without timestamps, a sequence is indistinguishable from a contradiction |
| Observations differ and carry **different methodologies** | They are not commensurable. Present both with their characterizations; combining them manufactures precision that no source supports |

---

## Part 5 — Wrong-answer archetypes

These are categories, not phrasings. Each has a narrow legitimate use; outside it, they're filler.

| Archetype | Why it's offered | Its only legitimate use |
|---|---|---|
| **Add capacity** — bigger window, bigger model, more resources | Plausible against any degradation | When the scenario establishes the resource is genuinely exhausted *and* nothing can be removed |
| **Add emphasis** — stronger wording, more insistent instruction | Feels responsive | Never, once a violation rate is on the table |
| **Add a cleanup stage** — post-process, normalize, filter afterwards | Feels safe and incremental | When the upstream genuinely cannot be changed, and no cost constraint forbids it |
| **Add a component** — a classifier, a validator model, an extra pass | Feels rigorous | When no cheaper in-band check exists and no complexity constraint forbids it |
| **Add annotation** — a certainty score, a flag, metadata about the problem | Resembles a solution | For **routing**. Never as a substitute for preventing the problem |
| **Retry harder** — blanket retry, more attempts, backoff everywhere | Correct for one failure class only | Transient failures, bounded, inside the tool |
| **Silently reconcile** — adjust values so they agree, resolve the conflict automatically | Produces clean output | Never. It destroys the only evidence that something was wrong |
| **Discard the inconvenient** — drop older data, filter low-certainty findings, require corroboration | Produces a tidy result | Never, when the discarded item carries information about *why* sources differ |
| **Flatten to uniformity** — one common format for everything | Sounds like good engineering | When no source's format carries meaning. Rare |
| **Redo everything** — reprocess the whole set, start from scratch | Guarantees correctness | When the successful portion is small or cannot be identified |
| **Relocate without rescoping** — reorganize, reorder, restructure the same content | Improves readability | When readability was the stated problem |
| **Build bespoke** — custom integration, custom infrastructure | Feels thorough | When nothing suitable exists |
| **Enumerate more cases** — extra branches, extra categories, longer checklists | Feels thorough | When the set is genuinely closed and stable |

A reliable tiebreaker: **prefer the option that changes the interface or structure over the one that changes the wording, and over the one that adds a stage downstream.**

---

## Part 6 — Discriminators for genuinely close pairs

When two options survive everything above, these are the distinctions the exam is actually testing.

| Pair | Decided by |
|---|---|
| Enforce in the tool **vs.** enforce at an interception hook | Do you own the tool's internals, and is the rule intrinsic to what it does? If yes, in the tool. If you must redirect a call into something you don't control, intercept |
| Permission denial **vs.** hook | Is the requirement *prohibit* or *perform*? Denial prohibits; a hook performs or inspects |
| Constrain the value **vs.** supply an identifier | Is the ambiguity about *form* or about *which one*? |
| Split a tool **vs.** clarify its description | Are the wrong calls *malformed for the operation*, or *the wrong operation chosen*? Split for the first; describe for the second |
| Force a specific action **vs.** force any action | Must a specific step come first, or do you merely need a structured result whose shape depends on the input? |
| Resume **vs.** branch **vs.** restart with a summary | Continuing one line of work; comparing alternatives without cross-contamination; or prior results are now actively wrong and must be removed rather than argued with |
| Always-loaded guidance **vs.** pattern-scoped **vs.** on-demand | Is it relevant always, when touching a kind of file, or when doing a kind of task? |
| Inline composition **vs.** conditional loading | Does the content need to be *assembled* from parts, or *withheld* until relevant? Only the second reduces cost |
| Plan first **vs.** act directly | Has localization already been done by available evidence? If yes, act |
| Plan first **vs.** elicit requirements | Is the unknown *which code is affected* or *what correct behaviour even is*? Planning explores what exists; it cannot surface a requirement nobody recorded |
| Instructions **vs.** examples | Is the model performing the wrong *procedure*, or producing the wrong *product*? Instructions direct process; examples demonstrate output and transfer judgment |
| Make a field optional **vs.** add examples | Is the information genuinely absent from the source, or present but in an unfamiliar form? |
| Risk-based routing **vs.** random sampling | Allocating scarce inspection across everything, or measuring the tier you stopped inspecting? Both are correct — for different jobs |
| Protocol error **vs.** result error | Was the call valid? Severity of the failure is irrelevant |
| Deferred batch **vs.** immediate | Compare the deadline against the worst-case processing window, per population |

---

## Part 7 — Fallback ordering

When nothing above resolves it, apply in order. The first that applies usually decides.

1. **Eliminate every option violating a stated constraint** — cost, latency, fixed component, enforcement location, scope.
2. **Eliminate every option whose mechanism class is too weak** for the stated requirement strength.
3. **Eliminate every option whose claimed cause doesn't match the symptom's signature** (absence vs. degradation vs. truncation; immediate vs. gradual).
4. **Eliminate every option that resolves only part of the listed symptoms.**
5. **Eliminate every option that silently alters, discards, or reconciles data.**
6. **Among survivors, prefer the one that makes the failure structurally impossible** over one that discourages or detects it.
7. **Prefer the fix at the point of decision** — the interface the model actually reads — over one upstream or downstream of it.
8. **Prefer separation** when one component is carrying two opposed objectives.
9. **Prefer the explicit over the inferred** — stated assumptions, carried metadata, computed decisions over raw signals.

---

## The invariants

Everything above reduces to these. They are domain-independent.

1. **Nothing is shared implicitly.** If a later step needs earlier information, something must place it there.
2. **A probabilistic mechanism cannot satisfy a binary requirement.**
3. **Enforce where it cannot be routed around** — and for terminal guarantees, outside the thing that terminates.
4. **Structure guarantees shape, never meaning.**
5. **A representation must be able to express every true state**, including absent, indeterminate, multiple, and contested.
6. **Compression destroys precision.** Exact values and metadata need their own channel.
7. **Only retry what is derivable** from what was already supplied.
8. **Surface discrepancies; never resolve them silently.**
9. **Separate opposed objectives**, and separate the author from the reviewer.
10. **Aggregates conceal segments**, and unmeasured tiers stay unmeasured.

---

See also: [Overview and cross-cutting principles](00-Overview-and-Priorities.md) · [D1](Domain-1-Agentic-Architecture-Orchestration/summary.md) · [D2](Domain-2-Tool-Design-MCP-Integration/summary.md) · [D3](Domain-3-Claude-Code-Configuration-Workflows/summary.md) · [D4](Domain-4-Prompt-Engineering-Structured-Output/summary.md) · [D5](Domain-5-Context-Management-Reliability/summary.md)
