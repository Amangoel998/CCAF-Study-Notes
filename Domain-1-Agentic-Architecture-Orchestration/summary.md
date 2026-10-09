# Domain 1 — Agentic Architecture & Orchestration

**Blueprint weight: 27% (~16 of 60 questions) · 60 questions in the bank, ~40 distinct concepts**

> The bank duplicates heavily — several scenarios appear three or four times reworded. Subtopic counts below are question counts; the concept counts are lower. See [1.0 Cross-Topic & Domain-Level](../../categorized_exam_dump/domain-1-agentic-architecture-orchestration/1.0-cross-topic-and-domain-level.md) for the duplicate clusters.

> **The one sentence that governs this domain:** the model decides *what* to do next; your code decides *whether it may* and *what it can see*. Nothing passes between steps unless you pass it.

Every question in this domain is really asking you to place a responsibility on one side of that line. If the question is "how does the agent know to do X next?", the answer is almost always *the model reasoned about it from context*. If the question is "how do we guarantee X happens / never happens?", the answer is almost always *code outside the model* — a tool-side check, a hook, a permission rule, or a post-loop check in the orchestration layer.

---

## 1.1 Agentic loops

### What an agentic loop actually is

An agentic loop is a `while` loop in **your** application code. It is not a feature of the model. The structure is always the same:

1. You send a `messages` array plus a `tools` array to the Messages API.
2. Claude responds. The response has a `stop_reason`.
3. If `stop_reason == "tool_use"`, you extract the `tool_use` block(s), execute the real function yourself, and append a `tool_result` block back onto `messages`.
4. You call the API again with the now-longer `messages` array.
5. You repeat until `stop_reason` is something terminal — `"end_turn"`, `"max_tokens"`, `"stop_sequence"`.

```python
while True:
    response = client.messages.create(model=..., messages=messages, tools=TOOLS)
    messages.append({"role": "assistant", "content": response.content})

    if response.stop_reason != "tool_use":
        break                      # "end_turn" and friends = we're done

    results = []
    for block in response.content:
        if block.type == "tool_use":
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": run_tool(block.name, block.input),
            })
    messages.append({"role": "user", "content": results})
```

### The mechanism the exam tests: reasoning, not routing

The single most repeated question in this subtopic is the warranty-claim scenario:

> The agent calls `lookup_order` and gets back order details showing the item was purchased **45 days ago**. How does the loop decide whether to call `process_refund` or `escalate_to_human` next?

The answer is: **the order details are added to the conversation as a `tool_result`, and the model reasons over them to choose the next action.** That's it. There is no planner, no decision tree, no router.

The three distractors are each a different way of pretending the intelligence lives somewhere else:

| Distractor | Why it's wrong |
|---|---|
| "The agent executes the remaining steps of a sequence planned at the start" | Agentic loops are *reactive*. The model did not and could not plan step 3 before seeing the result of step 2. If the sequence were known up front you wouldn't need an agent — you'd write a script. |
| "The agent follows a pre-configured decision tree mapping order attributes to tool calls" | That's a rules engine. It's legitimate software, but it is not what an agentic loop is, and it cannot handle cases the author didn't enumerate. |
| "The orchestration layer automatically routes to the next tool based on the order's status field" | The orchestration layer is dumb on purpose: it executes tools and shuttles results. It does not infer business policy from a `status` field. |

**Mental model:** the loop is plumbing. The model is the only thing in the system that makes a decision.

### The mechanism the exam tests: `stop_reason`, not heuristics

> What is the primary mechanism your application uses to decide whether to continue the loop or stop?

**Check the `stop_reason` field on each API response. Continue while it equals `"tool_use"`; exit when it becomes `"end_turn"` or another terminal value.**

**The exam keys two values. The live API returns more.**

| Value | Meaning | Exam-keyed? |
|---|---|---|
| `"tool_use"` | Claude wants to call tools — **continue** | ✅ |
| `"end_turn"` | Claude has finished — **terminate** | ✅ |
| `"pause_turn"` | A long-running server-tool turn — continue | Production only |
| `"max_tokens"` | Response hit the token ceiling | Production only |
| `"stop_sequence"` | A stop sequence was emitted | Production only |
| `"refusal"` | The model declined, on an otherwise-normal 200 response | Production only |
| `"model_context_window_exceeded"` | The response filled the context window — handle like `max_tokens` | Production only |

> **Production rule: treat anything other than `"end_turn"` as "not finished — check why," rather than assuming `"tool_use"`.**

### Where loops actually break

Step 3 is the failure point: **tool results must be appended to the conversation history.** Miss that and Claude never sees what the tool returned, so it has nothing new to reason about on the next iteration.

### The three anti-patterns for loop termination

The guide names these explicitly, and the exam draws distractors from all three.

| Anti-pattern | Why it fails |
|---|---|
| **Parsing natural language** — checking whether Claude said "I'm done" or "task complete" | Ambiguous. Claude may say "I've finished analysing the first file" while intending to continue. `stop_reason` exists precisely to remove this ambiguity |
| **An iteration cap as the *primary* stopping mechanism** | Either cuts off useful work (the task needed 12 turns) or runs pointless ones. Acceptable as a **safety net**, never as the control |
| **Checking content type** — `response.content[0].type == "text"` | Claude returns text **alongside** `tool_use` blocks. "Let me look up your order" in position [0] makes the loop exit mid-task |

**The premature-termination bug** is the content-type check, and the fix is always to check `stop_reason` — never to add an iteration cap. **Caps address runaway loops, not premature exits.** The exam offers caps as a plausible fix for premature termination specifically to catch this.

> ⚠️ **Additional trap: forcing `tool_choice: "any"` to stop the agent returning text.** This forces tool use even when the agent is genuinely finished — creating an **infinite loop**. Let the model signal completion naturally via `stop_reason`.

The remaining distractors are plausible-sounding substitutes that break in real conditions:

- **"Set `tool_choice` to `"none"` after the final expected tool call."** `tool_choice` is an *input* control that shapes what Claude may do on the next turn. It is not an output signal, and "the final expected tool call" presupposes you know the sequence — which you don't.
- **"Check whether the response contains a text content block."** Brittle. A single assistant message can contain a text block *and* a `tool_use` block in the same `content` array — Claude narrating "Let me look up that order" before calling the tool. You'd exit one turn early every time it narrates.
- **"Track the number of tool calls and exit at a preconfigured maximum."** A turn cap is a *safety net*, not a completion signal. It fires on runaway loops; it tells you nothing about whether the task finished. Most tasks finish well before it, and you'd hang waiting.

You should still have a `max_turns` ceiling — see 1.4, where what you do *after* it trips becomes its own exam question.

### Exam traps for 1.1

- Any answer implying the tool sequence was planned in advance.
- Any answer putting business logic in the orchestration layer.
- Confusing the loop's *stop condition* (`stop_reason`) with its *safety ceiling* (`max_turns`).
- Using a content-type check, natural-language parsing, or an iteration cap as the primary control.
- Adding an iteration cap to fix **premature** termination — caps fix runaway loops.
- Forcing `tool_choice: "any"` to suppress text — it prevents the agent ever finishing.
- Forgetting that tool results must be **appended to history** for the next iteration to use them.

---

## 1.2 Multi-agent orchestration

### The shape of the system: hub-and-spoke

The exam tests **one** orchestration pattern. A **coordinator** sits at the centre; **subagents** are the spokes.

> **The cardinal rule: ALL communication flows through the coordinator. Subagents never communicate directly with each other** — not for efficiency, not for convenience, not for any reason.

This centralisation buys three things the exam cares about:

| Benefit | Why |
|---|---|
| **Observability** | Every message is logged in one place |
| **Consistent error handling** | The coordinator applies uniform recovery policies |
| **Controlled information flow** | The coordinator decides what context each subagent sees |

> In current Claude Code a subagent *can* spawn its own subagents (nested delegation), so the absolute is an exam simplification. **On the exam, direct subagent-to-subagent communication is always the wrong answer.**

### The coordinator's four responsibilities

| # | Responsibility |
|---|---|
| 1 | **Dynamic subagent selection** — analyse the query and pick which subagents to invoke. **Do not route everything through the full pipeline** |
| 2 | **Research scope partitioning** — assign distinct subtopics or source types to minimise duplication (one agent searches academic papers, another news) |
| 3 | **Iterative refinement loops** — evaluate output for gaps, re-delegate with targeted queries, re-invoke until coverage suffices. **Not a single-shot process** |
| 4 | **Centralised routing** — all communication through the hub |

### The narrow decomposition failure — trace failures to their origin

> ⚠️ **This is the exam guide's sample Question 7 and a recurring pattern.**

> A research system is asked about "renewable energy technologies." The report covers solar and wind thoroughly but says nothing about geothermal, tidal, biomass, or fusion. **Every subagent performed well** — search returned relevant results for every query it received; synthesis accurately combined everything it was given.

**Root cause: the coordinator decomposed the topic into only two subtopics.** The missing categories were never assigned to anyone.

| Proposed fix | Why it fails |
|---|---|
| The search agent's queries were too narrow | It searched thoroughly for **what it was assigned** |
| Synthesis failed to spot the gaps and request more | Possible as an enhancement, but the gap originated upstream |
| The document analysis agent lacked sources | No agent was ever told to look |
| **Add more subagents** | They receive equally narrow assignments |

> **The diagnostic rule: if the output is incomplete in *scope* (not depth), the coordinator's decomposition is almost always the root cause.** Do not blame the subagent that produced the output — check whether the coordinator gave it the right input.

```
                 ┌──────────────┐
   user ───────► │ coordinator  │ ◄──── holds the request, relays everything
                 └──┬────┬───┬──┘
          Task ─────┘    │   └───── Task
                    Task │
        ┌───────────┐ ┌──┴────────┐ ┌───────────┐
        │ web search│ │ doc analys│ │ synthesis │   ← separate contexts, no links
        └───────────┘ └───────────┘ └───────────┘
```

### Parallelism is a message-structure property

> The coordinator invokes the web search subagent, waits for the response, then invokes the document analysis subagent. The two tasks are independent — neither needs the other's output. Latency is 40+ seconds.

**Fix: structure the coordinator so it emits both `Task` tool calls in a *single response message*.** Multiple `tool_use` blocks in one assistant message are dispatched concurrently. Spreading them across two conversation turns forces serialization, because turn 2 can't begin until turn 1's `tool_result` comes back.

The distractors are all things engineers actually reach for:

- **"Switch to Haiku to reduce execution time."** Makes each leg faster; the legs are still sequential. You've optimized the wrong axis.
- **"Build an async layer outside the agent with parallel threads."** Now the coordinator doesn't know what's running. You've lost its visibility, its error handling, and its ability to react to partial results.
- **"Add system prompt instructions about running things in parallel."** Prompting may encourage the model to emit both calls in one message, but the *enabling mechanism* is the message structure. The exam wants the mechanism, not the nudge.

### When delegation does not happen at all

> The coordinator's output says "I'll ask the web search agent to find sources" — but no subagent ever runs. No errors in the logs. The coordinator continues as though delegation happened.

**Cause: `"Task"` is missing from the coordinator's allowed-tools configuration.** The model can *talk about* delegating all day — that's just text generation. Actually spawning a subagent requires invoking the `Task` tool, and a tool that isn't in the tool list cannot be invoked.

The diagnostic signature matters:

| Symptom | Cause |
|---|---|
| Reasons about delegation, no invocation, **no errors** | `Task` not in allowed tools |
| Partial / malformed `tool_use` blocks in logs | `max_tokens` truncation |
| Subagent runs but arrives with nothing useful | Context not forwarded (that's 1.3) |
| Never even reasons about delegation | System prompt never described the available subagents |

"No error, no call, but confident narration" is the fingerprint of a missing tool.

### Don't delegate what you already know

> Follow-up queries like "summarize what we learned about market trends" take 40+ seconds. The coordinator spawns a synthesis subagent and passes it 80K+ tokens of accumulated findings — findings the coordinator already holds in its own context from orchestrating the research.

**Fix: have the coordinator handle straightforward summarization directly**, reserving subagent spawning for complex analysis that genuinely needs specialization or isolation.

Distractors that are all local optimizations of an unnecessary operation:

- *Enable prompt caching on the synthesis subagent* — cheaper, still slow, still unnecessary.
- *Spawn it with reduced context and let it request findings on demand* — more round trips, not fewer.
- *Pre-generate cached summaries at multiple granularities* — a whole caching subsystem built to avoid noticing you didn't need the subagent.

### Dynamic routing beats fixed routing

> Simple factual queries ("What year was the Paris Climate Agreement signed?") traverse all four subagents sequentially (40+ seconds). Complex comparative research genuinely benefits from the full pipeline. The query distribution is **diverse and evolving**.

**Fix: let the coordinator analyze each query and dynamically decide which subagents to invoke.** The decisive property is that the input distribution is **not stable** — which rules out every fixed-category answer by construction:

- A trained complexity classifier goes stale as patterns shift, and needs labelled data plus a retraining pipeline.
- A hard-coded fast path for one category works until an input doesn't fit the category.
- Pattern-based routing with predefined categories has the same brittleness with extra config.

Whenever a scenario establishes that the set of inputs keeps changing, any fixed enumeration of that set is wrong — the model is already a general-purpose classifier.

The model is already a general-purpose classifier. Use it.

### Closing the loop: gap detection and re-delegation

> The document analysis agent reports "sources discuss API authentication but lack details on token refresh patterns." In a strict one-way pipeline (search → analyze → synthesize), this insight arrives after the search phase has already completed, so nothing can act on it.

**Fix: have the analysis agent report *specific named gaps* to the coordinator; the coordinator launches targeted searches for those gaps and re-invokes analysis; repeat until coverage criteria are satisfied.** This converts a one-shot pipeline into a reflection loop.

Note precisely why the near-miss answers fail:

- **Add a research-planning agent before search.** Planning happens before any evidence exists. It cannot anticipate a gap that is only discoverable *after* analysis.
- **Have synthesis attach confidence scores and flag low-confidence sections for manual review.** This labels the incomplete report instead of completing it. Symptom, not cause.
- **Have the coordinator watch for general gap indicators and re-run search without re-running analysis.** You get new sources but never verify the gap actually closed. The loop has no termination criterion.

A real reflection loop needs all three parts: **specific gap identification → targeted re-delegation → re-verification.**

### Exam traps for 1.2

- **Blaming a downstream subagent for a coverage gap** when the coordinator's decomposition was too narrow.
- **Adding more subagents to fix a decomposition problem** — they get equally narrow assignments.
- **Proposing direct subagent-to-subagent communication** as an efficiency improvement.
- Assuming subagents share memory or inherit the coordinator's history.
- Treating parallelism as a prompt instruction rather than a message structure.
- Building infrastructure outside the coordinator and losing its visibility.
- Choosing a static classifier/router when the scenario says patterns evolve.
- Accepting a reflection loop that re-searches but never re-checks.

---

## 1.3 Subagent invocation and context passing

This is the second-densest subtopic in Domain 1 (12 questions) and the single biggest source of trap answers on the whole exam.

### The `Task` tool and `allowedTools`

`Task` is how a coordinator spawns subagents. **Current Claude Code renamed it `Agent`; `Task` still works as an alias. Answer "Task tool" on the exam, expect `Agent` in current code.**

> **Hard requirement: the coordinator's `allowedTools` must include `"Task"` (or `"Agent"`).** It is a **binary gate**, not a soft preference — without it the coordinator physically cannot spawn subagents, no matter how they are defined.

*(Current SDK docs describe `allowedTools` as an auto-approve list: leaving `Agent` off doesn't remove the tool, it sends every spawn through the permission callback, which denies it in an unattended run. Same outcome, different mechanism.)*

Each subagent is an **AgentDefinition** with three fields:

| Field | Purpose |
|---|---|
| **Description** | What it does — used by the coordinator to decide **when to invoke it** |
| **System prompt** | The instructions it follows |
| **Tool restrictions** | Which tools it can access, scoped to its role |

### The three rules of context passing

| # | Rule |
|---|---|
| **1** | **Include complete findings from prior agents** — in full, in the prompt. Do not assume a downstream agent can "look up" prior results. It cannot |
| **2** | **Use structured formats that separate content from metadata** — the claim *and* the source URL, document name, page number |
| **3** | **Specify goals, not procedures** — what to achieve and what quality criteria to meet, not step-by-step instructions |

### The attribution failure — a specific exam pattern

> A synthesis agent produces an excellent report with **no source attribution**. The web search subagent returns results with URLs and titles. The document analysis subagent returns analysis with page references. **Both subagents are verified working correctly.**

**Root cause: the coordinator passed content but stripped the metadata.** The synthesis agent **literally cannot cite sources it was never given.**

```json
{ "findings": [{
    "claim": "Solar panel efficiency has increased 25% in the last decade",
    "source_url": "https://example.com/solar-report",
    "document_name": "Annual Solar Industry Report 2024",
    "page_number": 14,
    "confidence": "high",
    "retrieved_by": "web_search_agent" }] }
```

| Proposed fix | Why it fails |
|---|---|
| Modify the synthesis agent's **prompt** to cite sources | It has no source information to cite. Instructing it invites fabricated citations |
| Give synthesis **direct tool access** so it can re-run queries | Duplicates work, and the coordinator still isn't passing metadata |
| Blame the **output format** the search agent used | The search agent returned the metadata; the coordinator dropped it |

**Fix: require the coordinator to pass structured metadata alongside content, preserving source URL, document name, and page number for every finding.**

### The rule: subagents share nothing

Each subagent gets a fresh context window containing exactly what the coordinator put in its prompt. It cannot see:

- the coordinator's conversation history,
- other subagents' transcripts, results, or tool calls,
- anything from a previous invocation of itself.

> The web search and document analysis subagents both finish successfully. The coordinator invokes the synthesis subagent, which replies: **"I cannot complete the task because no research findings were provided."**

**Cause: the coordinator never put the earlier outputs into the synthesis prompt.** Fix: explicitly include them.

The distractors here are the most-tested trap family on the exam. Learn the *diagnostic* that kills each one:

| Distractor | Why it's wrong | The giveaway |
|---|---|---|
| "Synthesis needs tools to fetch other agents' histories" | No such retrieval mechanism exists. Agent transcripts are not an addressable store unless you built one. | — |
| "Synthesis's context window is too small" | A too-small window produces **truncation** — partial findings, dropped early content, degraded detail. It does not produce **"no findings were provided."** | Total absence ≠ overflow |
| "The subagents need to share an API connection" | Transport has nothing to do with context. Two agents over one HTTP connection still have two independent `messages` arrays. | Connection ≠ context |

**Burn this in:** *complete absence of data = missing explicit hand-off. Degraded or partial data = context pressure.* The exam uses these two failure modes interchangeably in the options and expects you to tell them apart from the symptom.

### Sequential hand-off: the coordinator relays

> The web search agent has gathered relevant sources. The document analysis agent now needs to examine them.

**The coordinator receives the search agent's output and includes the relevant findings in the prompt when it invokes the analysis agent.** Not a message queue, not direct agent-to-agent invocation, not an automatically shared memory store. The coordinator is the bus.

### Scaling the hand-off: reference IDs + a shared store

Inline injection works fine until the payload gets big. When combined findings run to 50K+ tokens, stuffing them into every downstream prompt is wasteful and may not fit.

**Fix: pass *reference identifiers* and configure the downstream subagent with read access to a shared memory store where the other subagents deposited their results.** The subagent retrieves exactly the records it needs.

The key qualification is **"configure with read access"** — that access is something you set up, not something that exists by default. An option saying the subagent "automatically accesses the shared memory store" is wrong for the same reason everything else in 1.3 is wrong.

Choose by payload size:

| Situation | Pattern |
|---|---|
| Small outputs, few agents | Inline the findings in the prompt |
| Large outputs (tens of thousands of tokens) | Reference IDs + explicitly configured shared store |
| Never | Message queues with subscriptions; direct agent-to-agent calls |

### Instruct with goals and quality criteria, not procedures

> The coordinator gives the web search subagent detailed step-by-step instructions: exact queries, source priorities, date filters. Three production problems appear: (1) when searches fail it reports "insufficient results" instead of trying alternatives, (2) quality drops on emerging topics, (3) it rarely surfaces valuable tangential sources.

**Fix: specify research goals and quality criteria — coverage breadth, source diversity, recency — and let the subagent determine its own search strategy.**

All three symptoms share one root cause: a prescribed procedure has no branch for "the procedure didn't work." Note how carefully the distractors are tuned:

- **"Add fallback directives for alternative query formulations."** Still prescriptive. You've enumerated plan B; plan C still fails.
- **"Remove the procedural details entirely and just give a simple goal."** Over-corrects. Now nothing enforces coverage, diversity, or recency — you lose the quality bar along with the rigidity.
- **"Classify topics as well-defined vs. exploratory and use different instruction sets."** Added machinery that preserves the rigidity inside each branch.

The correct answer keeps the *criteria* (what good output looks like) and drops the *procedure* (how to get there). That distinction — **criteria yes, steps no** — generalizes across the whole exam.

### Subagents as a context-degradation remedy

> An agent has spent 25 minutes exploring a rendering subsystem — shader code, `VulkanPipeline`, `FrameGraph`. Its recent responses have started referencing "typical rendering patterns" instead of the specific classes it discovered earlier. The engineer now asks about physics engine integration with rendering.

That shift from *specific names* to *generic patterns* is the textbook symptom of context degradation.

**Fix: summarize the key rendering findings (concrete class names, architectural patterns), then spawn a subagent for the physics exploration with that summary in its initial context.** The subagent investigates physics in a clean window but still knows the rendering facts it must integrate with.

Why the alternatives lose:

- *Spawn the physics subagent independently and synthesize manually afterwards* — it explores physics blind to the rendering architecture it has to integrate with.
- *`/clear` and start fresh from CLAUDE.md paths* — throws away 25 minutes of discovered specifics.
- *Keep going, but write more targeted prompts naming the classes* — the window is still crowded with irrelevant shader detail; precision keeps slipping.

The same pattern recurs in a payment-processing variant (8 of 45 files read, responses less accurate, test files being missed): **spawn subagents for specific bounded questions** ("find the test files," "trace the refund flow") while the main agent coordinates findings and preserves the high-level picture. Each subagent burns its own context, not the coordinator's.

> **Resuming an interrupted subagent** used to be filed here because the scenario features a subagent. The decision it tests is resume-and-announce-the-delta, so it now sits in [1.7](#17-session-state-and-resumption).

### Exam traps for 1.3

- Picking "bigger context window" or "shared API connection" when the symptom is total absence of data.
- Assuming any automatic sharing — of history, memory stores, or results.
- Assuming subagents can invoke each other.
- Keeping procedural instructions when the symptom is poor adaptation.
- Resuming without announcing what changed.

---

## 1.4 Workflow enforcement and handoff

### The enforcement spectrum

Two fundamentally different ways to enforce ordering:

| | Mechanism | Reliability |
|---|---|---|
| **Prompt-based guidance** | "Always verify the customer's identity before processing a refund" | **Probabilistic** — works ~90–95% of the time |
| **Programmatic enforcement** | A hook, prerequisite gate, or code-level check that **physically blocks** the downstream tool | **Deterministic** — works every time |

### The exam's decision rule

| Operation class | Mechanism |
|---|---|
| **Financial** (refunds, transfers, payments) | **Programmatic.** One unverified refund is a financial loss |
| **Security** (identity verification, access control) | **Programmatic.** One bypass is a breach |
| **Compliance** (AML, regulatory) | **Programmatic.** One missed check is a legal penalty |
| **Low-stakes** (formatting, style, ordering) | Prompt-based is fine — an inconsistency is not a business risk |

> **The exam will present prompt-based solutions as options for high-stakes scenarios. Reject them.** Enhanced system prompts, few-shot examples, and stronger instructions all improve probability but **none provides a deterministic guarantee.** When the scenario involves money, security, or compliance, the answer is always programmatic enforcement.

One more distractor worth naming: **a routing classifier**. A classifier decides *which agent* handles a request; the compliance failure happens *within* an agent's execution sequence. **Classifiers handle routing, not per-agent workflow enforcement.**

### Prerequisite gates in practice

> Production data shows refunds processed without verifying account ownership in **8% of cases**. The system prompt clearly instructs verification first. It works 92% of the time.

**Fix: a programmatic prerequisite gate.** Before `process_refund` executes, code checks that `get_customer` has returned a verified customer ID **in the current session**. If not, the call returns: *"Cannot process refund — customer identity not verified. Please call get_customer first."*

**The gate is code, not a prompt instruction.** Even if the model decides to skip verification, the gate blocks the call and forces the correct order. This takes 8% to **0%** — not by improving the prompt, but by making the wrong order impossible.

### The enforcement hierarchy

This subtopic has exactly one idea applied to five scenarios: **put the guarantee where it cannot be skipped.** Rank the mechanisms by how hard they are to bypass:

| Mechanism | Strength | Use for |
|---|---|---|
| Server-side logic **inside the tool** | Strongest — the model cannot route around it | Business rules, money limits, authorization |
| `permissions.deny` | Strong — blocked at the permission layer | Hard prohibitions on paths and tools |
| Hooks (`PreToolUse` / `PostToolUse`) | Strong — deterministic code on every call | Interception, mechanical transformation |
| Orchestration-layer checks **around** the loop | Strong — runs regardless of how the loop ended | End-state guarantees |
| System prompt / CLAUDE.md / few-shot examples | Advisory — "mostly works" | Style, tone, preferences, conventions |

The exam's favourite move is to show you a prompt-based approach that is working *almost* well enough and ask what to do. **A 97% compliance rate is a 3% compliance failure, and compliance requirements are binary.** The answer is always to move the rule down to a harder layer — never to write the instruction more emphatically.

### The $500 reimbursement rule

> Company policy: reimbursements above $500 require manager approval before disbursement. The agent handles hundreds of requests daily. How do you ensure the threshold cannot be bypassed?

**Fix: enforce the threshold inside the reimbursement tool itself.** Under $500 → disburse and return success. Over $500 → create a pending approval request and return a status indicating manager review is pending. The tool is a server-side boundary; the model literally cannot get money out of it another way.

> **Filing note:** the bank now files this under **2.1 Tool Interface Design**, because the correct answer changes the tool's interface rather than the workflow around it. It is kept here because it is the cleanest illustration of the enforcement ladder — and because the exam will not tell you which subtopic a scenario belongs to.

Why each alternative leaks:

- **System prompt instruction plus an `approved_by_manager` parameter.** The model fills in that parameter. You have asked the component you don't trust to certify its own trustworthiness.
- **Two tools: `auto_reimburse` (capped at $500) and `manager_approval`.** Better, but the model still chooses which tool to call. Choice is the vulnerability.
- **A `PreToolUse` hook that sets a `requires_approval` flag.** A hook *can* enforce this, but it is a client-side configuration — misconfigure or omit it and the rule vanishes. A business rule about disbursing money belongs on the server that disburses it.

(Contrast with the 1.5 compliance scenario, where a hook *is* the right answer. The distinction: there, the requirement is "intercept the model's call and redirect it"; here, the rule is intrinsic to what the tool itself does. When the question offers tool-side enforcement of a rule about the tool's own behaviour, take it.)

### Guaranteeing the interaction ends resolved or escalated

> Complex billing disputes require 6+ tool calls. The agent sometimes exhausts its `max_turns` limit after gathering data but before resolving or escalating. The requirement: **every** interaction must end in resolution or escalation, regardless of how the loop terminated.

**Fix: an orchestration-layer check that runs after the loop exits.** If the agent's outcome is neither resolved nor escalated, your code programmatically calls `escalate_to_human` with the accumulated context and tool results.

The point is that this check lives **outside the model's turn budget**, so running out of turns cannot prevent it. Every distractor is still inside the budget:

- *A prompt instruction to escalate when few turns remain* — requires the agent to reach that reasoning step and obey it, using a turn it may not have.
- *A `PreToolUse` hook that counts invocations and terminates at 80% of the limit* — the hook fires, but then hands control back to the model to do the escalating.
- *Split into two sequential agents with separate budgets* — doubles the budget, guarantees nothing about how the second agent ends.

**Generalize this:** a guarantee about *how a process ends* must be implemented by something that outlives the process.

### Escalation criteria

> What are the right triggers for escalating to a human?

**Escalate when: (1) the customer explicitly requests a human, (2) the issue requires a policy exception beyond the agent's authorization, or (3) the agent cannot make meaningful progress.** These combine customer intent with operational limits and leave room for judgment on novel cases.

- *A rules engine mapping issue types to escalation* removes judgment and misses anything not enumerated.
- *Sentiment analysis on a frustration score* is a proxy. A calm customer with an unauthorized request still needs a human; a frustrated customer with a solvable problem doesn't.
- *After N consecutive failed tool calls* is arbitrary — too early for a genuinely multi-step issue, too late for one that was instantly out of scope.

### Structured handoff

> The agent handles a billing dispute, calls `get_customer` and `lookup_order`, identifies a promotional pricing error requiring manager approval, and must now escalate.

**Fix: compile a structured handoff — customer details, order information, the identified issue — before calling `escalate_to_human`.**

The investigation already happened. Discarding it means the human repeats it, which is the entire cost the agent was supposed to save.

- *Pass only the customer's original message* — forces the human to redo everything.
- *Persist the full history to a database and pass a reference ID* — now the human has to go fetch it; adds latency and a lookup step for data that fits inline.
- *Attempt the refund anyway and escalate only if the system rejects it* — attempting an unauthorized action in order to discover it's unauthorized.

### Structured handoff

> ⚠️ **The critical constraint: the human agent does NOT have access to the conversation transcript.** They cannot scroll back through the chat. **The summary is the only information they receive.**

**Five required fields:**

| Field | Why |
|---|---|
| **Customer ID** | So the human can pull up the account |
| **Conversation summary** | What was asked, and what has been attempted |
| **Root cause analysis** | The agent's assessment of the underlying issue |
| **Refund amount** (if applicable) | The **specific figure**, not a vague reference |
| **Recommended action** | What the agent believes should happen next |

If the summary is incomplete, the human must ask the customer to repeat everything — which is the entire cost the agent was meant to save.

**Trap: handoff summaries that omit the customer ID or the recommended action.**

### Multi-concern requests

> "I want to return my order, update my shipping address, and ask about my loyalty points."

**The correct approach: decompose → investigate in parallel using shared context → synthesise a unified resolution** addressing all items in one response. The customer's account information is relevant to all three, so it's gathered once.

**Wrong: handling them sequentially in separate conversations, or addressing only the first and forgetting the rest.**

### Fixing misidentification at the source

> A delete-contact tool handles requests like "delete the duplicate Acme Corp." The CRM contains "Acme Corp," "Acme Corporation," and "ACME Corp Inc." **8% of deletions are reversed within 24 hours** because the wrong record was deleted.

**Fix: present the matched records with their differentiating fields and require a single-click confirmation of the intended target before executing.**

This addresses the actual defect — ambiguous identification — at the moment the ambiguity exists, at minimal friction cost. Compare:

- *Require an exact CRM record ID instead of natural language* — destroys the usability that made the tool worth building, and the user still has to identify the right record, now without help.
- *Deploy automated duplicate detection and merging* — solves a different problem (that duplicates exist) and is far more invasive.
- *Soft delete with 30-day recovery* — a safety net. The 8% misidentification rate is unchanged; you've only made the consequences cheaper. Useful, but mitigation, not a fix.

General principle: **when a scenario gives you an error *rate*, prefer the answer that lowers the rate over the answer that softens the consequence.**

### Exam traps for 1.4

- Choosing stronger prompt wording for an absolute requirement.
- Letting the model select between a "safe" tool and an "unsafe" tool as the control.
- Putting an end-state guarantee inside the loop it is supposed to survive.
- Choosing recovery/mitigation when the question asks you to reduce an error rate.

---

## 1.5 Agent SDK hooks

### The two hook directions, and what each returns

Hooks inject **deterministic** behaviour into an otherwise probabilistic system, sitting at the boundary between the model's decisions and the real world.

| Hook | Runs | Can block? | Returns | Purpose |
|---|---|---|---|---|
| **`PreToolUse`** | **Before** the tool executes | **Yes** | `permissionDecision`: `allow` / `deny` / `ask` / `defer`, plus optional `updatedInput` to rewrite the arguments | **Policy enforcement** |
| **`PostToolUse`** | **After** the tool executes, but **before the model processes the result** | Stops the loop, but cannot undo the side effect | `updatedToolOutput` to replace what the model sees | **Data normalisation** |

> `updatedMCPToolOutput` is the older field, MCP-only, now deprecated. `updatedToolOutput` covers built-in and MCP tools alike.

**Memorise the direction.** `PreToolUse` enforces policy *before* execution; `PostToolUse` transforms data *after* execution. The exam tests this constantly.

### Purpose 1: `PostToolUse` for data normalisation

This is a major documented use that is easy to miss if you only think of hooks as formatters.

> A support agent uses three tools. `get_customer` returns Unix timestamps and numeric status codes. `lookup_order` returns ISO 8601 strings and English statuses. `check_shipping` returns `DD/MM/YYYY` and single-character codes (`S` shipped, `P` pending).

Without normalisation the model must interpret three date formats and three status representations **on every iteration**. It parses a Unix timestamp correctly once and misreads it the next time; it confuses day/month order; it reads `P` as "processed" instead of "pending."

**Fix: a `PostToolUse` hook that normalises every result before the model sees it** — all dates to ISO 8601, all status codes to human-readable strings, currency to a consistent decimal format with currency code.

**Trap: relying on the model to normalise heterogeneous formats itself.** That is exactly what produces the inconsistency. Normalisation belongs in deterministic code.

### Purpose 2: `PreToolUse` for policy enforcement

`PreToolUse` hooks are the implementation mechanism for the prerequisite gates in 1.4. Three documented use cases:

| Use case | Mechanism |
|---|---|
| **Threshold enforcement** | Intercept `process_refund`; if the amount exceeds the limit, block and redirect to human escalation. The tool never executes |
| **Prerequisite gates** | Intercept `transfer_funds`; if the required AML check hasn't completed this session, block and return an error directing the agent to complete it first |
| **Approval workflows** | Intercept `approve_discount` above a threshold; pause and route to a manager approval queue. The tool executes only after approval |

> ⚠️ **The headline trap of this subtopic:** the exam presents a **`PostToolUse` hook as a solution for blocking** a policy-violating action. It is wrong by construction — `PostToolUse` runs *after* execution, so by the time it fires the non-compliant action has already happened. "Flag the completed transfer and queue it for manual review" is detection, not prevention.

### The decision framework

| Requirement | Mechanism | Nature |
|---|---|---|
| Must be followed **100%** of the time | **Hooks** | Deterministic |
| Preferred, but occasional deviation acceptable | **Prompts** | Probabilistic |

Apply it by consequence, not by confidence:

- Would the business **lose money** from a single failure? → hook.
- Would the business face **legal risk** from a single failure? → hook.
- Is it a formatting or style preference? → prompt is fine.

Worked comparison from the guide:

| Scenario | Prompt | Hook |
|---|---|---|
| International transfers must pass AML checks | ~95% compliance; the 5% is a **regulatory violation** | `PreToolUse` blocks `transfer_funds` until `aml_check` passes — 100% |
| Responses should use markdown | Works most of the time; no business risk | Unnecessary overhead |
| Refunds above $500 need approval | A single failure is a large unapproved refund | Intercept, check, block, escalate — 100% |

**The decision is never whether prompts are "good enough." It is whether the consequence of a single failure justifies a deterministic guarantee.**

### Beyond tool hooks: `PreCompact`

> The guide scopes 1.5 to `PreToolUse`/`PostToolUse`, but the same mechanism covers session events, and the practice bank has tested `PreCompact`.

`PreCompact` runs immediately before Claude Code compacts the conversation — whether the user typed `/compact` (matcher **`manual`**) or the context hit the auto-compact window (matcher **`auto`**). It receives the transcript path, the trigger, and any custom instructions passed to `/compact`, so it can **archive the full transcript before the summary discards detail**.

**Exit code `2`, or `"decision": "block"` in its JSON output, blocks the compaction.** A `PostCompact` event fires afterwards carrying the generated summary.

The pattern generalizes: **a `Pre` event can inspect and block; a `Post` event can only observe what already happened.**

### The three-requirement sorting question

> A team has three requirements, all currently written in CLAUDE.md:
> 1. Never modify the `db/migrations/` directory.
> 2. Prefer the custom logging module over `console.log`.
> 3. Auto-format TypeScript with Prettier after every edit.
>
> Claude has already violated #1.

**Correct allocation:**

| Requirement | Mechanism | Why |
|---|---|---|
| Never touch `db/migrations/` | `permissions.deny: ["Edit(./db/migrations/**)"]` | Absolute prohibition → hard boundary |
| Prefer the custom logger | Leave it in CLAUDE.md | Style preference; needs judgment; occasional deviation is acceptable |
| Run Prettier after edits | `PostToolUse` hook on `Edit|Write` | Mechanical, deterministic, must happen 100% of the time |

The question is really "can you tell a prohibition from a preference from a transformation?" The distractors fail by applying one mechanism to all three:

- *Stronger CLAUDE.md wording plus few-shot examples for everything* — prompt-level guidance for a requirement that has already been violated.
- *Hooks for all three* — a hook to enforce a logging *preference* is over-engineering; preferences should tolerate judgment, and `permissions.deny` is cleaner for the prohibition.
- *Path-scoped `.claude/rules/` for everything* — rules files are still loaded as guidance (see 3.3). Adding a config layer does not create enforcement.

### "Emphasis isn't working" is always a hook question

> CLAUDE.md says "use 4-space indentation, always run Prettier." Code reviews show 30% of files have inconsistent formatting. Adding emphatic language reduces violations to 15%, but never to zero.

**Fix: configure a `PostToolUse` hook with an `Edit|Write` matcher that automatically runs Prettier on each file Claude modifies.**

30% → 15% is the exam telling you *the mechanism is wrong, not the wording*. Halving a violation rate through emphasis proves the lever is weak; pulling it harder gets you 7.5%, not 0%.

- *Path-scoped `.claude/rules/` files* — still prompt-level; the model still has to remember.
- *A `Stop` hook that prompts Claude to check its formatting* — asks the model to fix it, costing a turn and still depending on compliance.
- *A dedicated formatting skill with detailed examples* — still requires the model to invoke and follow it on every single edit.

### Hard business rules at the interception point

> Compliance requires that refunds exceeding $500 escalate to a human — it cannot be left to model discretion. The system prompt says so clearly. Production logs show the agent occasionally processes high-value refunds directly: a **3% failure rate**.

**Fix: implement a hook that intercepts tool calls; when `refund_process` carries an amount over $500, block it and invoke human escalation independently of the model's judgment.**

3% is not "pretty good" for a compliance control; it's an audit finding. The distractors:

- *Modify the refund tool to return an error message for amounts over $500* — the agent can keep calling it, re-interpret the error, or attempt a workaround. It adds friction rather than a boundary. (Compare the 1.4 reimbursement question, where tool-side enforcement that *creates a pending approval record* is correct. Returning a bare error is not the same as implementing the policy.)
- *Few-shot examples showing escalation at various amounts* — still advisory; still relies on model memory.
- *More emphatic system prompt language* — the exact thing that already failed.

### Exam traps for 1.5

- Reading a *reduced* violation rate as progress rather than as proof the mechanism is wrong.
- Using a hook for a soft preference, or a prompt for a hard rule.
- Confusing `PreToolUse` (can block) with `PostToolUse` (runs after, for transformations).

---

## 1.6 Task decomposition strategies

The guide names **two** decomposition patterns and **one** failure mode. Everything in this subtopic is one of the three.

| Pattern | Use when | How to recognize the condition |
|---|---|---|
| **Fixed sequential pipeline (prompt chaining)** | The steps are known in advance and don't change with intermediate results | A fixed set of checks that all run on every input |
| **Dynamic adaptive decomposition** | The plan must evolve as the agent discovers more | The target is unknown and must be located by following evidence; scope is open-ended |

| Task characteristic | Pattern |
|---|---|
| Steps known in advance, structured input | Fixed pipeline |
| Open-ended, unknown scope | Dynamic |
| Multi-file code review | Fixed pipeline |
| Legacy codebase exploration | Dynamic |
| Document extraction (fields predetermined) | Fixed pipeline |
| Debugging an unfamiliar system | Dynamic |

**Trade-offs, as the guide frames them:**

| | Fixed pipeline | Dynamic |
|---|---|---|
| Strength | Consistent, reliable, easy to debug and monitor — the same input always follows the same path | Adapts to the problem; discovers and responds to unexpected complexity |
| Weakness | **Cannot adapt.** If step 2 finds something that should change step 3, it can't | Less predictable; execution time varies; harder to debug |

> **Trap:** matching the pattern to what *sounds more sophisticated* rather than to the task. The exam offers a fixed pipeline for open-ended investigation, and dynamic decomposition for structured processing.

*(The "map → prioritize → revise" approach to an open-ended codebase is an instance of dynamic decomposition, not a third pattern.)*

### Fixed steps → prompt chaining

> A code review assistant analyzes every PR on three fixed aspects: code style compliance, potential security issues, documentation completeness.

**Fix: prompt chaining — break the review into sequential steps where each aspect is analyzed separately, then combine the outputs in a final synthesis step.**

Every PR gets all three checks. Nothing varies. That's a chain.

- *A single comprehensive prompt covering all three* — loses the benefit of focused sub-analysis; attention splits across concerns and recall drops on each (the same effect 4.6 exploits).
- *Orchestrator-workers with dynamic task determination* — **the key over-engineering trap.** Orchestrator-workers exists to decide *which* subtasks to run. Here you already know: always these three. There is nothing to orchestrate.
- *Routing by PR type* — adds a branch where no branching is needed.

### Unknown failure path → dynamic subtasks

> An API endpoint intermittently returns 500 errors. The codebase has 200+ files. The developer doesn't know which components are involved; the cause could be in routing, middleware, business logic, or the database layer.

**Fix: have the agent dynamically generate investigation subtasks based on what it discovers at each step, adapting its exploration plan as the error path emerges.**

Debugging is inherently evidence-driven — step 3 depends on what step 2 found.

- *A fixed sequence (grep logs → read handlers → check DB → examine middleware)* — spends effort on layers the evidence already ruled out, and has no branch for "none of this found it."
- *Parallel worker agents covering all four layers simultaneously* — broad coverage, no prioritization, four streams of mostly-irrelevant findings to reconcile. "Parallel" is right when subtasks are independent (1.2); it's wrong when each step should inform the next.
- *A comprehensive plan mapping all code paths before exploring* — you cannot exhaustively plan 200+ files, and the plan is obsolete after the first finding.

### Open-ended scope → map, prioritize, revise

> Add comprehensive tests to a legacy codebase: 200 files, minimal coverage, and the engineer hasn't specified which modules to prioritize.

**Fix: use Glob and Grep to map the codebase structure, identify heavily-coupled modules, create a prioritized plan for high-impact areas, and revise the plan as dependencies are discovered.**

The three moves are **survey cheaply → rank by impact → stay revisable**.

- *A fixed testing schedule based on directory structure* — directory layout is not a proxy for risk or complexity.
- *Read all 200 files before writing any tests* — burns the entire context before producing value; the inventory can emerge as testing progresses.
- *Start with the first module alphabetically* — no risk-based prioritization at all.

### The attention dilution problem

A named failure mode the guide tests directly: **an agent processing too many items in a single pass produces inconsistent depth.**

**Symptoms — learn to recognize these:**

- Detailed analysis for the first few items, increasingly shallow for later ones.
- **The same pattern flagged as a problem in one item and approved in another.**
- Obvious bugs missed in some items while minor style issues are caught in others.

> A review agent processes **14 files**. Files 1–5 get detailed feedback with line references. Files 6–9 get moderate feedback. Files 10–14 get superficial feedback that misses null-pointer bugs and SQL injection. A `forEach` loop is flagged as inefficient in File 3 while identical code in File 11 draws no comment.

**Why:** the model allocates attention across all items in context. More items means less attention each; early items get a disproportionate share and later ones get skimmed.

**Fix: multi-pass architecture — two layers.**

1. **Per-item local passes.** Analyse each file in its own pass, with the full attention budget on one item.
2. **A cross-item integration pass.** After the local passes, one separate pass looks across all items for cross-cutting concerns: data-flow issues, inconsistent pattern usage, cross-file dependencies.

The local passes catch local issues consistently; the integration pass catches relationships that no single-item pass can see.

**The traps, all four of which the exam uses:**

| Proposal | Why it fails |
|---|---|
| A more powerful model | **Architectural problem, not a capability problem.** Single-pass over many items produces uneven depth regardless of model |
| A larger context window | Same — the items all fit; attention is what's divided |
| A stronger prompt emphasising thoroughness | Improves average quality; does not change attention allocation |
| **Batching into groups of 5** | Reduces dilution *within* each batch but **misses cross-batch issues**. Without a dedicated integration pass, data-flow problems and pattern inconsistencies across batches go undetected |

That last one is the subtle distractor: batching is half the fix. **Per-item passes without a cross-item pass is an incomplete answer.**

### Exam traps for 1.6

- Reaching for orchestrator-workers when the subtasks never vary.
- Choosing exhaustive upfront analysis on a large codebase.
- Choosing parallel fan-out when each step should inform the next.
- Applying a fixed pipeline to open-ended investigation, or dynamic decomposition to structured processing.
- Treating attention dilution as a model or context-size problem.
- Accepting batching **without** a cross-item integration pass.

---

## 1.7 Session state and resumption

The largest subtopic in the domain (20 questions). It splits into two halves: **API-level state** (what the Messages API remembers — nothing) and **Claude Code session management** (resume, continue, fork).

### The API is stateless

> An agent runs a multi-step identity verification before resetting passwords. After the customer answers the **third** verification question, the agent asks for their name again — as if the earlier exchange never happened.

**Cause: the Claude API is stateless. Prior conversation turns must be re-sent in the `messages` array with every request.** If the application omitted them, the information does not exist from the model's point of view.

Distractors and their tells:

- *"Claude's memory is limited to two turns by default."* There is no such default. Context is bounded by tokens, not turn count.
- *"The prompt lacks instructions to remember earlier answers."* Memory comes from message history, not from being told to remember. No instruction can recover a message you didn't send.
- *"The verification tool is clearing conversation state."* Tool invocation does not touch the `messages` array. Your application owns it.

This single fact underlies the whole domain: if a later step needs earlier data, **something in your code must put it there.**

### Statelessness, part two: a current-state object

> Users refine search criteria mid-conversation: "raise the budget to $650K", "actually, prefer a condo not a house." The assistant sometimes continues referencing the *original* preferences even though the updates are in the history. Context usage is only at **35%**.

"Only 35%" is the exam explicitly removing context pressure as a cause. The real problem is that the model is being asked to reconstruct current state by reading a chronology and inferring which statement supersedes which.

**Fix: maintain a structured state object with the *current* preferences, update it on each change, and include it in every request.** One authoritative source; no inference required.

```json
{"current_criteria": {"budget_max": 650000, "property_type": "condo", "bedrooms": 2}}
```

- *Prune the turns containing outdated preferences* — discards history (including why things changed) and the model still has to infer from what's left.
- *Few-shot examples of correct preference updating* — guides the parsing; doesn't replace it.
- *System prompt emphasis on honouring the most recent preference* — still relies on correct chronological parsing, which is exactly the step that is failing.

**Generalization:** whenever a fact must be *current* rather than *historical*, hoist it out of the conversation into a structured block you re-send each turn. Same pattern as Domain 5's persistent case-facts block.

### The three session management options

The guide frames this as exactly three options, each doing a different job.

| Option | What it does | Use when | Do **not** use when |
|---|---|---|---|
| **`--resume <name>`** | Restores the **entire** history — every tool result, analysis, and reasoning chain | Prior context is still valid; files haven't changed | **Files have been modified** — see the stale context problem |
| **`fork_session`** | Branches an **independent copy** from a shared baseline; branches can't see each other | You've completed an analysis and want to explore divergent approaches from it | You simply want to continue. **Fork is for divergence, not continuation** |
| **Fresh start + summary injection** | A new session seeded with a curated summary of prior findings | Tool results are stale; context has degraded over a long session | Prior context is valid and you want the full history — resume is more efficient |

**How sessions get their names:** `--resume` only resumes sessions that **already exist** — it takes an ID or name, or opens an interactive picker, but it **never creates** one. You name a session when you start it with `--name` / `-n`, or mid-session with `/rename`. `-c` / `--continue` resumes the most recent conversation in the current directory without naming anything.

**How forking is actually expressed:** in the SDK, `fork_session` is set **beside** `resume`, not instead of it — `resume` picks the session, and `fork_session: true` starts a new session from a *copy* of that history rather than appending to it. The CLI pairs `--fork-session` with `--resume` the same way. So the choice is **append or branch**, not one option or the other.

### Flag reference

| Flag | Behaviour |
|---|---|
| `--name <name>` / `-n` | Names a session **when you start it** |
| `/rename` | Names it mid-session |
| `--resume <id\|name>` | Resumes an **existing** named session (or opens a picker) |
| `-c` / `--continue` | Resumes the most recent conversation in this directory |
| `--fork-session` (with `--resume`) | Branches a copy instead of appending |

> An engineer spent two hours yesterday in a session named `auth-deep-dive` investigating authentication flows. She has worked on three other codebases since then.

**`--resume auth-deep-dive`.** `--continue` would grab whichever session was most recent — one of the three unrelated ones. Starting fresh and re-reading the same files wastes two hours of accumulated context.

### The stale context problem — the central concept of 1.7

> ⚠️ **The official guide and the practice bank disagree here. Follow the guide.**

**How it manifests:** a developer analyses a codebase, modifies three files, and resumes. Claude gives **contradictory advice** — recommending changes that were already made, referencing code that no longer exists, sometimes citing the old version and sometimes the new one.

**Why:** resuming restores the *entire* history, including every tool result. A file read last session is still in the conversation with its **old contents**. The model reasons from that stale data alongside any fresh read.

**The naive fix, and why the guide rejects it:** resuming and asking the agent to re-read the modified files is *better than nothing*, but **the stale tool results remain in the history**. The model may still reference the old information, especially for tangential decisions that don't directly involve the changed files.

**The guide's correct fix: start a fresh session with a structured summary of prior findings, and specify which files changed so the agent performs targeted re-analysis of just those.** No stale tool results; the knowledge is preserved without the outdated data.

```
Fresh session
  ├─ Injected summary: "Prior analysis found X, Y, Z across the codebase."
  ├─ "The following 3 files have changed: auth.ts, database.ts, api-routes.ts"
  └─ Agent re-reads only those 3, combines with the preserved summary
```

**Targeted re-analysis vs. full re-exploration:** re-reading 50 files because 3 changed is waste. Name the changed files; the summary covers everything else.

| Approach | Verdict |
|---|---|
| **Fresh session + summary + name the changed files** | ✅ **The guide's answer.** No stale results, knowledge preserved, minimal re-reading |
| Resume and ask it to re-read the changed files | ⚠️ The **bank's** answer, and explicitly called out by the guide as *not the best* — stale results persist |
| Fresh session, re-analyse everything from scratch | ❌ Wasteful; discards prior findings |
| `fork_session` to branch around the change | ❌ **The fork inherits the stale context.** Forking copies the history, stale results included |

> **If both "resume and inform" and "fresh start + summary + inform" appear as options, choose the fresh start.** The bank contains at least one question keyed the other way; the official guide is the better authority for the exam.

### Decision matrix

| Situation | Choice |
|---|---|
| Continuing yesterday's work, **no files changed** | `--resume` |
| Comparing two refactoring approaches | `fork_session` |
| Resuming after modifying 3 of 50 files | **Fresh start + summary** |
| Long session with cluttered, degraded history | **Fresh start + summary** |
| Exploring a testing strategy vs. a documentation strategy | `fork_session` |
| Resuming after dependency updates | **Fresh start + summary** (many files may have changed indirectly) |

### The same rule applies to subagents

> A subagent spent 30 minutes analyzing a legacy payment system (47 files) and documenting data flows. The session was interrupted. Meanwhile a teammate merged a PR renaming two utility functions.

The presence of a subagent is scenery — the decision is identical to the single-session case. **The bank keys "resume and tell it about the renames"; the guide's principle says a fresh session seeded with the findings plus the list of changes is cleaner, because the resumed transcript still holds the pre-rename file contents.**

What is unambiguous either way:

- Never resume **silently** — the agent keeps reasoning with obsolete symbol names.
- Never start fresh **without the findings** — that repeats 30 minutes of discovery.
- **Resuming never re-reads files on its own.** A transcript is a frozen snapshot; any delta is yours to announce.

### Stale tool results: fresh session + structured summary

> A customer returns **4 hours** after a 32-turn billing-dispute session. That transcript contains a `lookup_order` result reading `Status: PENDING, Expected resolution: 24-48 hours`. In testing, the agent keeps saying "your refund is still being processed" **even after fresh tool calls return different information.**

That last clause is the crux: the stale result is still sitting in context, competing with the fresh one, and the model has no principled way to adjudicate between them.

**Fix: start a new session, inject a structured summary of the previous interaction (issue type, actions taken, resolution status), then make fresh tool calls before engaging the customer.** Continuity is preserved as a *summary*; the data is re-fetched fresh; the stale payload is simply gone.

- *Resume the full history but filter out previous `tool_result` messages* — removes the stale payloads but also the issue context woven through those turns, and the surrounding assistant messages still restate what the results said.
- *Resume with an instruction to prefer the most recent results* — stale and fresh data sit side by side and the instruction has to win an attention contest on every turn.
- *Resume and auto-call every previous tool at session start* — adds latency, and the old results are still in context next to the new ones.

**Rule: when old tool output is actively wrong, remove it. Don't try to out-argue it with instructions.**

### Forking for independent branches

> Yesterday's analysis of a legacy auth module identified two refactoring approaches: extract a microservice, or refactor in place. The engineer wants to explore **both** deeply — proposing concrete code changes for each — before deciding which to implement.

**Fix: use `fork_session` to create two branches from yesterday's analysis and explore one approach in each.** Both branches inherit the full analyzed context; neither can contaminate the other.

- *Resume and explore both sequentially in one thread* — **the key trap.** The first approach's reasoning sits in context while the second is evaluated, biasing the comparison and mixing assumptions. When the task is to *compare* options fairly, shared reasoning is a defect, not a feature.
- *Resume for the first, new session with manual context for the second* — asymmetric: one branch has rich context, the other a reconstruction. Not a fair comparison.
- *Two fresh sessions seeded with a summary of findings* — repeats the discovery work and loses detail.

### The decision table

| Situation | Choice |
|---|---|
| Continue a specific past investigation, nothing changed | `--resume <name>` |
| Continue the most recent thing | `-c` / `--continue` |
| **Some files changed since last time** | **Fresh session + summary + name the changed files** |
| Compare two approaches fairly from shared context | **Fork** (`--fork-session` with `--resume`) |
| Old tool results are now wrong | **New session + structured summary + fresh tool calls** |
| A fact must be *current*, not historical | Structured state object re-sent every request |
| Crash recovery mid-pipeline | Structured per-agent export + coordinator manifest |

### Exam traps for 1.7

- Choosing `--continue` when the scenario mentions intervening unrelated work.
- Assuming a resumed session re-reads changed files on its own.
- **Choosing `--resume` after files have been modified** — stale tool results persist in history.
- **Using `fork_session` to escape stale context** — the fork copies the history, stale results included.
- Confusing fork with resume: fork **branches a copy** (original untouched); resume **appends** to the same session.
- Full re-exploration of a large codebase when only a few files changed.
- Sequential exploration in one thread when the task is an unbiased comparison.
- Trying to suppress stale context with instructions instead of removing it.
- Persisting a raw transcript for crash recovery instead of structured state plus a manifest.

---

## The five sentences that answer most of Domain 1

1. **Nothing is shared implicitly.** The API is stateless; subagents have no shared memory. If a later step needs earlier data, your code must put it there.
2. **Prompts advise; code enforces.** When a rule is absolute, the answer is a tool-side check, a hook, a permission rule, or an orchestration check — never stronger wording.
3. **Total absence ≠ degradation.** "No findings were provided" is a missing hand-off. "Vague, generic, forgetting specifics" is context pressure. Different symptoms, different fixes.
4. **Criteria, not procedures.** Tell a subagent what good output looks like; let it choose how to get there.
5. **Guarantee at the boundary.** Anything that must be true when the process *ends* belongs to code that outlives the process.

## Study order within this domain

1. **1.7 session management** — densest (20 questions); memorize the resume/continue/fork table and the consequences of statelessness.
2. **1.3 context passing** — second densest (12), and its trap family (bigger window / shared connection / automatic sharing) recurs throughout the exam.
3. **1.4 + 1.5 enforcement** — one idea in many disguises; learn to rank mechanisms by bypassability.
4. **1.2 orchestration** — parallel `Task` calls in one message, and the gap-detection reflection loop.
5. **1.1 + 1.6** — small but cheap points. Know `stop_reason`, and know prompt chaining vs. orchestrator-workers.
6. **[1.0 cross-topic questions](../../categorized_exam_dump/domain-1-agentic-architecture-orchestration/1.0-cross-topic-and-domain-level.md)** — eight authored scenarios that don't announce their subtopic, which is how the real exam presents them.

## Links

[Domain 1](https://claudecertificationguide.com/learn/1-agentic-architecture) ·
[1.1 Agentic loops](https://claudecertificationguide.com/learn/1-agentic-architecture/1-1-agentic-loops) ·
[1.2 Orchestration patterns](https://claudecertificationguide.com/learn/1-agentic-architecture/1-2-orchestration-patterns) ·
[1.3 Subagent invocation & context](https://claudecertificationguide.com/learn/1-agentic-architecture/1-3-subagent-invocation-context) ·
[1.4 Workflow enforcement & handoff](https://claudecertificationguide.com/learn/1-agentic-architecture/1-4-workflow-enforcement-handoff) ·
[1.5 Agent SDK hooks](https://claudecertificationguide.com/learn/1-agentic-architecture/1-5-agent-sdk-hooks) ·
[1.6 Task decomposition](https://claudecertificationguide.com/learn/1-agentic-architecture/1-6-task-decomposition) ·
[1.7 Session state & resumption](https://claudecertificationguide.com/learn/1-agentic-architecture/1-7-session-state-resumption)
