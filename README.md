# Claude Certified Architect — Foundations Study Materials

Visual study pack for the Claude Certified Architect — Foundations certification. Covers all five exam domains with diagrams, a cheat sheet, and worked code examples.

## What's in here

- **[cheatsheet.md](cheatsheet.md)** — single-page reference of flags, paths, enum values, and distractor-traps. Lock these in first.
- **[diagrams/](diagrams/)** — 17 architectural diagrams, one concept per file, each with annotations and anti-patterns.
- **[examples/](examples/)** — full runnable code for agent loops, hooks, MCP tools, and extraction pipelines.

## The exam at a glance

| Parameter | Value |
|---|---|
| Question format | Multiple choice, 1 correct of 4 |
| Scoring | 100–1000 scale, passing score **720** |
| Guessing penalty | None — answer every question |
| Scenarios | **4 randomly picked from 6** |
| Target candidate | Solution architect with 6+ months hands-on Claude experience |

The exam tests **tradeoff judgement**, not memorisation. Questions almost always ask "which is **most effective**" — meaning two or three answers are plausible, and you must pick the one that addresses the root cause with proportionate effort.

## Domains and weights

| # | Domain | Weight | Core focus |
|---|---|---|---|
| 1 | Agentic Architecture & Orchestration | **27%** | Agent loops, coordinator/subagent, hooks, `Task` tool, sessions |
| 2 | Tool Design & MCP Integration | **18%** | Tool descriptions, `isError`, `tool_choice`, `.mcp.json` |
| 3 | Claude Code Configuration & Workflows | **20%** | CLAUDE.md hierarchy, `.claude/rules/`, skills, plan mode, CI/CD |
| 4 | Prompt Engineering & Structured Output | **20%** | Few-shot, JSON schemas, validation loops, batch API |
| 5 | Context Management & Reliability | **15%** | Lost-in-the-middle, escalation, provenance, confidence calibration |

## The six scenarios

Each scenario frames a set of questions around a realistic production context. You'll see 4 of these on exam day.

1. **Customer Support Resolution Agent** — returns, billing disputes, account issues; MCP tools for customer/order lookup and refunds; 80%+ first-contact resolution target.
2. **Code Generation with Claude Code** — slash commands, CLAUDE.md, plan mode vs direct execution, refactoring and debugging workflows.
3. **Multi-Agent Research System** — coordinator agent delegating to search, document-analysis, synthesis, and report-generation subagents with citations.
4. **Developer Productivity with Claude** — exploring unfamiliar codebases, generating boilerplate, built-in tools (Read/Write/Bash/Grep/Glob) plus MCP.
5. **Claude Code for Continuous Integration** — automated code review, test generation, PR feedback, minimising false positives.
6. **Structured Data Extraction** — unstructured documents to JSON, schema validation, handling edge cases, integrating with downstream systems.

## Recommended study path

1. **Read the cheat sheet end to end.** The exam rewards concrete knowledge of flag names, file paths, and enum values. Most wrong answers are plausible-sounding inventions.
2. **Walk through the diagrams in weight order** — Domain 1 (1–7), Domain 3 (8–11), Domain 4 (12–14), Domain 2 (15), Domain 5 (16–17). Each file is self-contained and takes ~10 minutes.
3. **Run the worked examples** alongside each diagram. Feeling the API shapes by hand is the fastest way to internalise them.
4. **Work the practice questions** in the official exam guide and the blog sources below, *after* you can predict which anti-pattern each distractor represents.

## Key principles the exam keeps testing

- **Deterministic beats probabilistic** when business rules are at stake — hooks beat prompts.
- **Descriptions drive tool selection** — fix the description before adding classifiers, routing layers, or few-shot examples.
- **Explicit context beats implicit inheritance** — subagents don't see what the coordinator saw; pass it in the prompt.
- **Loop on `stop_reason`, not on text content** — `end_turn` is the only reliable completion signal.
- **Structure errors, don't flatten them** — generic "operation failed" destroys the coordinator's ability to recover.
- **Attention dilutes with volume** — larger context windows don't fix multi-file review; multi-pass does.

## Sources

- Official exam guide — *Claude Certified Architect – Foundations Certification Exam Guide*, v0.1 (Feb 2025)
- Claude API, Agent SDK, Claude Code, and MCP public documentation
- Community study notes — [paullarionov/claude-certified-architect](https://github.com/paullarionov/claude-certified-architect)
