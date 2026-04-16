# Cheat Sheet

Single-page reference. If you forget something going into the exam, it's almost certainly here.

---

## Stop reasons (Claude API)

| Value | Meaning | Action |
|---|---|---|
| `end_turn` | Model finished its turn | **The only reliable loop-exit signal** |
| `tool_use` | Model wants to call a tool | Execute, append result, loop |
| `max_tokens` | Hit the `max_tokens` cap | Increase limit; response is truncated |
| `stop_sequence` | Hit a configured stop sequence | App-specific handling |

**Distractor traps:** parsing assistant text for "done"/"complete" · using `max_iterations=N` as the primary stopping mechanism · treating *absence* of tool calls as completion.

---

## `tool_choice` values

| Value | Behaviour | When to use |
|---|---|---|
| `{"type": "auto"}` | Model chooses tool or returns text | Default |
| `{"type": "any"}` | Must call *some* tool (any of them) | Guarantee structured output |
| `{"type": "tool", "name": "X"}` | Must call tool `X` | Force a specific first step (e.g. `extract_metadata` before enrichment) |

---

## CLAUDE.md hierarchy

```
~/.claude/CLAUDE.md          ← USER     — personal, NOT shared via VCS
.claude/CLAUDE.md   OR       ← PROJECT  — shared with team, in VCS
CLAUDE.md (repo root)
<subdir>/CLAUDE.md           ← DIRECTORY — applies when editing files in that subtree
```

**Import syntax:** `@./standards/testing.md` — relative or absolute paths, max nesting depth **5**.

**Classic diagnostic:** a teammate doesn't get your instructions → you put them in `~/.claude/CLAUDE.md` (user-level) instead of `.claude/CLAUDE.md` (project-level).

---

## `.claude/rules/` — path-scoped rule files

```yaml
---
paths: ["**/*.test.tsx", "**/*.test.ts"]
---

Tests must use describe/it blocks. Use factories, not hardcoded fixtures.
```

- Loads **only** when editing files matching a glob in `paths`
- Prefer over subdirectory CLAUDE.md when conventions span many directories (e.g., test files scattered throughout the codebase)
- Prefer subdirectory CLAUDE.md when conventions are genuinely directory-local

---

## Claude Code CLI flags (CI/CD)

| Flag | Purpose |
|---|---|
| `-p` / `--print` | Non-interactive mode — **required for CI**, prompt → stdout → exit |
| `--output-format json` | Emit structured JSON output |
| `--json-schema '…'` | Enforce a schema on output (use with `--output-format json`) |
| `--resume <session-name>` | Continue a named prior session |

**Distractors that don't exist:** `CLAUDE_HEADLESS=true` env var · `--batch` flag · redirecting stdin from `/dev/null` (workaround, not the documented way).

---

## Slash commands (built-in)

| Command | Purpose |
|---|---|
| `/memory` | Open CLAUDE.md for edit; verify which memory files are loaded |
| `/compact` | Compress conversation history — frees context, but may lose exact values/dates |

---

## Custom commands & skills

```
.claude/commands/review.md         ← PROJECT slash command (shared via VCS)
~/.claude/commands/review.md       ← USER slash command (personal)
.claude/skills/review/SKILL.md     ← PROJECT skill (modern format)
~/.claude/skills/review/SKILL.md   ← USER skill
```

**`SKILL.md` frontmatter:**

```yaml
---
context: fork                   # run in isolated subagent — output doesn't pollute main session
allowed-tools: ["Read", "Grep"] # restrict tool access for safety
argument-hint: "path to analyze" # prompt developer when invoked without args
---
```

- **Skill vs CLAUDE.md:** skills are on-demand invocation for specific tasks; CLAUDE.md is always-loaded project standards.
- **Personal variant:** copy a project skill to `~/.claude/skills/` under a **different name** so you don't shadow the shared one for teammates.

---

## MCP server configuration

| File | Scope | Use for |
|---|---|---|
| `.mcp.json` (project root) | Team, committed to VCS | Shared servers (GitHub, Jira, Slack) |
| `~/.claude.json` | User, NOT in VCS | Personal/experimental servers |

**Env var expansion** — keeps secrets out of VCS:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

Tools from **all** configured servers are discovered on connection and available simultaneously.

---

## MCP error response shape

**Good — structured:**
```json
{
  "isError": true,
  "content": {
    "errorCategory": "transient",
    "isRetryable": true,
    "message": "Orders API timeout after 5s",
    "attemptedQuery": "order_id=12345",
    "partialResults": null
  }
}
```

**Bad — uniform:** `{"isError": true, "content": "Operation failed"}` — strips the context the coordinator needs to decide whether to retry, reroute, or escalate.

---

## Error categories (structured error responses)

| Category | Example | `isRetryable` | Agent action |
|---|---|---|---|
| **Transient** | Timeout, 503, network flap | `true` | Retry, possibly with backoff |
| **Validation** | Bad input format, missing required field | `false` | Fix input and retry |
| **Business** | Policy violation, threshold exceeded | `false` | Explain to user; offer alternative |
| **Permission** | Access denied | `false` | Escalate |

**Critical distinction:** access failures (timeouts) and valid empty results (no matches) must be reported differently — same "empty response" shape confuses the coordinator.

---

## Agent SDK essentials

```python
agent = AgentDefinition(
    name="coordinator",
    description="Handles research requests across subagents",
    system_prompt="You are a coordinator...",
    allowed_tools=["Task", "get_customer", "lookup_order"],
    # ↑ "Task" is REQUIRED for the coordinator to spawn subagents
)
```

**Hooks — when to choose which:**

| Hook | Intercepts | Use for |
|---|---|---|
| `PreToolUse` | Outgoing tool call | Block policy-violating actions, redirect to escalation |
| `PostToolUse` | Tool result before model sees it | Normalise data formats, trim verbose fields |

**Hooks vs prompts:** hooks give **deterministic** guarantees (100%); prompts are **probabilistic** (>90%, not 100%). When failure has financial, legal, or safety consequences → hooks.

**Subagents — the four rules:**
1. Spawn via the `Task` tool (coordinator must have `"Task"` in `allowed_tools`)
2. **Do not inherit** parent context — pass explicitly in the prompt
3. Multiple `Task` calls in a single coordinator response → execute in parallel
4. All inter-subagent communication routes through the coordinator (for observability)

**Session control:**
- `--resume <name>` — continue a named session (risky if files changed since)
- `fork_session` — create an independent branch from a shared baseline (compare approaches)
- Starting fresh with a summary > resuming with stale tool results

---

## Message Batches API

| Attribute | Value |
|---|---|
| Cost saving | **50%** vs synchronous |
| Processing window | Up to **24 hours** — no latency SLA |
| Multi-turn tool calling | **Not supported** — one request = one response |
| Correlation | `custom_id` field links request to response |

**Use for:** overnight tech-debt reports, weekly audits, bulk extraction (10k+ docs).
**Never use for:** blocking workflows — pre-merge checks, interactive review, anything a human waits for.

**SLA math:** need a result in 30 hours with a 24-hour batch window → submit no later than 6 hours after the task starts.

---

## JSON schema design for extraction

- **Use `tool_use` with a schema** → eliminates syntax errors (not semantic errors — those need validation loops)
- **Nullable fields** (`"type": ["string", "null"]`) → prevents the model fabricating values when the source doesn't contain them
- **Enum + `"other"` + detail field** → extensible categorisation without losing data outside predefined categories
- **Enum includes `"unclear"`** → an honest "unclear" is better than a wrong category
- **Make fields required only when the information is always present** in the source — otherwise the model will invent values to satisfy the schema

---

## Escalation — valid vs invalid triggers

| ✓ Valid trigger | ✗ Invalid (distractor answer) |
|---|---|
| Explicit customer request ("get me a manager") | Sentiment analysis — mood ≠ complexity |
| Policy gap / policy silent on the specific case | Model-self-reported confidence score |
| Agent cannot make progress after a real attempt | Arbitrary complexity heuristic |
| Multiple customer matches → **ask for identifiers**, don't guess | Auto-classifier trained on ticket history (over-engineered for a first fix) |

**Key nuance:** expressing frustration ≠ requesting a human. Acknowledge emotion → offer a concrete resolution → escalate only if the customer reiterates the demand for a human.

---

## Built-in tools — quick pick

| Need | Tool | Example |
|---|---|---|
| Find files by name/extension | **Glob** | `**/*.test.tsx`, `src/**/*.ts` |
| Search content across files | **Grep** | Function name, error message, imports |
| Load a full file | **Read** | Inspect a module end-to-end |
| Create a new file | **Write** | Scaffold a new module |
| Targeted change via unique anchor text | **Edit** | Replace one function |
| Edit failed (non-unique match) | **Read** + **Write** | Load full file, modify, rewrite |
| Run shell command | **Bash** | git, npm, tests, builds |

**Exploration pattern:** don't read everything upfront. Grep for entry points → Read those files → Grep for their consumers → repeat.

---

## Plan mode vs direct execution

| Use plan mode when… | Use direct execution when… |
|---|---|
| Changes span dozens of files | Single-file fix with clear stack trace |
| Multiple valid approaches exist | Well-scoped, unambiguous change |
| Architectural decisions involved | Adding one validation check |
| Unfamiliar codebase | You already know what to do |
| Library migration affecting 45+ files | Bug fix with known root cause |

**Combined:** plan → user approves → direct execution of the approved plan.
**Explore subagent:** use for verbose discovery phases to keep the main context clean.

---

## Top distractor traps (the "almost right" wrong answers)

- ❌ `CLAUDE_HEADLESS=true`, `--batch` — **don't exist**
- ❌ Sentiment analysis for escalation routing
- ❌ Model's self-reported confidence (1–10) as routing signal
- ❌ Parsing assistant text to detect loop completion
- ❌ `max_iterations=N` as the *primary* stopping mechanism
- ❌ Larger context window to fix multi-file review attention issues → use multi-pass instead
- ❌ Building a routing classifier as the *first fix* for tool misselection → fix tool descriptions first
- ❌ Using prompt instructions to enforce critical business rules → use hooks
- ❌ Returning empty results marked as success when a tool actually failed
- ❌ Giving a subagent all tools "in case" → degrades selection reliability
- ❌ Merging all tools into one to eliminate ambiguity → over-engineered vs fixing descriptions
- ❌ Requiring developers to split large PRs → shifts burden, doesn't solve the system problem
- ❌ Self-review by the same instance that generated the code → use an independent instance

---

## One-sentence rules to live by

- **The only reliable loop exit is `stop_reason == "end_turn"`.**
- **Tool descriptions are the LLM's tool-selection mechanism** — fix them first.
- **Subagents don't inherit context** — pass it explicitly.
- **Hooks for deterministic rules, prompts for probabilistic guidance.**
- **Structure errors to enable recovery; don't flatten them.**
- **Summarise with sources preserved** — never drop `claim → source` mappings.
