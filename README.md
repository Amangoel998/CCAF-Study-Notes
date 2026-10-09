# CCA-F Study Notes

Study material for the **Claude Certified Architect – Foundations** exam — 60 questions, 120 minutes, five weighted domains.

Every blueprint subtopic is explained through the scenarios it's tested with: what the concept is, why the correct answer is correct, and **why each attractive distractor fails**. Content is cross-checked against the official lesson pages, with discrepancies between sources flagged inline.

| Path | Contents |
|---|---|
|  [01-Pattern-Decoder.md](01-Pattern-Decoder.md) | Domain-independent decision rules — how to read a scenario you've never seen |
  [02-Decision-Tables.md](02-Decision-Tables.md) | Every "given X, choose Y" table in one place, for last-pass revision |
## Coverage vs. blueprint

| Domain | Blueprint | Questions | Distinct concepts |
|---|---|---|---|
| D1 — Agentic Architecture & Orchestration | 27% | 60 | ~40 |
| D2 — Tool Design & MCP Integration | 18% | 53 | ~30 |
| D3 — Claude Code Configuration & Workflows | 20% | 30 | **~18** ← biggest gap |
| D4 — Prompt Engineering & Structured Output | 20% | 57 | ~33 |
| D5 — Context Management & Reliability | 15% | 36 | ~26 |
## How the exam asks questions

Almost every question is a **scenario with a diagnosis hidden in it**: a system, a symptom, often a number, and four plausible fixes — three of which are real engineering practices that are wrong *for this symptom*.

The skill tested isn't "do you know what a hook is." It's **can you read the symptom and identify which layer is broken**.

Two habits carry most of the weight:

- **Incidental detail is never incidental.** Any fact the scenario states but doesn't need is there to eliminate options — a resource well below its limit, a fix that *partly* worked, a forbidden cost, a component that must stay fixed.
- **Four failure signatures are not interchangeable.** *Total absence* = a missing hand-off. *Partial* = capacity. *Imprecise* = attention dilution. *Structurally broken* = an output ceiling. A capacity explanation predicts partial output, **never** total absence.

## The eight ideas that decide most questions

1. **Nothing is shared implicitly.** The API is stateless; subagents share no memory. If a later step needs earlier data, your code must put it there.
2. **Prompts advise; code enforces.** `tool-side logic > permissions.deny > hooks > orchestration checks >> prompts`. A 3% violation rate on a compliance rule is a finding, not a near-miss.
3. **Fix the interface, not the wording.** Fabricated parameters and wrong tool selection are schema and description problems. Prefer *structurally impossible* over *discouraged*.
4. **Structure ≠ correctness.** A schema guarantees shape, not meaning. Required fields *cause* fabrication; confidence fields never prevent it.
5. **Match the recovery to the error.** Transient → retry in the tool. Business → surface with a reason. Absent from the input → retrying cannot help.
6. **Preserve facts, compress prose.** Exact values live in a structured block re-sent every request. Prose is lossy for data.
7. **Criteria, not procedures — concrete, not descriptive.** Tell a subagent what good output looks like; show a judgment call rather than describing it.
8. **Separate things that pull against each other.** Finding vs. filtering, generating vs. reviewing, comparing two options — split them.


> Note: re-read the **"exam traps"** section closing each subtopic — they're the distilled failure modes.

## Sources

[Curriculum](https://claudecertificationguide.com/learn) · [Blueprint](https://claudecertificationguide.com/architect-foundations) · [Anthropic docs](https://docs.anthropic.com/) · [MCP specification](https://github.com/modelcontextprotocol)

Where the official guide and current product behaviour diverge, both are given — with the guide's answer marked as the one to use on the exam.
