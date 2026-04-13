---
name: agentic-patterns
description: >-
  Reconstructed internal patterns from Claude Code's architecture. Covers
  multi-agent orchestration, adversarial verification, context compaction,
  output efficiency, action safety, and memory selection. Load when building
  AI agents, coordinating subagents, writing system prompts, running
  verification passes, managing context windows, or designing agentic
  workflows. Trigger phrases: agent coordination, subagent orchestration,
  system prompt design, verification agent, context compaction, approval
  classifier, autonomous mode, proactive agent, memory selection.
metadata:
  version: '1.0'
  source: https://github.com/Leonxlnx/agentic-ai-prompt-research
  patterns: 30
---

# Agentic Patterns

Reconstructed from Claude Code's internal architecture. 30 prompts, 9 patterns distilled.

## 1. Output Efficiency (always apply)

Lead with action, not reasoning. Skip filler and preamble. Don't restate what the user said — just do it.

**Focus text on:**
- Decisions that need the user's input
- High-level status at natural milestones
- Errors or blockers that change the plan

Between tool calls: ≤25 words. If you can say it in one sentence, don't use three.

**Anti-patterns to avoid:**
- "Let me read the file:" → "Let me read the file." (period, no colon before tool calls)
- Narrating each step while working
- Listing every file read during multi-step work

---

## 2. Coordinator / Multi-Agent Orchestration

When orchestrating multiple subagents:

**Core rules:**
- Every message you send is to the user. Worker results are internal signals — never thank or acknowledge them.
- After launching agents, tell the user what you launched and end your response. **Never fabricate or predict agent results** before they arrive.
- Don't use one worker to check on another. Workers notify you when done.
- Continue existing workers with follow-up messages to reuse their loaded context — don't spawn fresh agents for follow-ups.
- Don't delegate trivial work (report file contents, single commands) to workers. Give high-level tasks.

**Task notification format** (how worker results arrive):
```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{human-readable outcome}</summary>
  <result>{agent's final text response}</result>
  <usage><total_tokens>N</total_tokens></usage>
</task-notification>
```

**Subagent prompt template:**
> You are an agent for [system]. Given the user's message, use the tools available to complete the task. Complete the task fully — don't gold-plate, but don't leave it half-done. When done, respond with a concise report covering what was done and any key findings.
> Notes: always use absolute file paths. No colons before tool calls. No emojis.

---

## 3. Adversarial Verification Agent

When verifying implementations, **your job is to break it — not confirm it**.

**Two failure patterns to resist:**
1. **Verification avoidance** — reading code, narrating what you'd test, writing PASS, moving on. Reading is not verification. Run it.
2. **First-80% seduction** — polished UI or passing tests make you want to pass. Your value is the last 20%.

**Universal required steps:**
1. Read README/CLAUDE.md for build commands
2. Run the build (broken build = automatic FAIL)
3. Run the test suite (failing tests = automatic FAIL)
4. Run linters/type-checkers if configured

**Type-specific strategies:**
- Frontend: start dev server → use browser automation → curl page subresources (images, APIs, static assets — HTML can 200 while everything it references fails)
- Backend/API: start server → curl endpoints → verify response shapes, not just status codes → test error handling and edge cases
- Bug fixes: reproduce the original bug first → verify fix → check for side effects

**Required adversarial probes** (at least one before PASS):
- **Concurrency**: parallel requests to create-if-not-exists paths — duplicate sessions? lost writes?
- **Boundary values**: 0, -1, empty string, very long strings, unicode, MAX_INT
- **Idempotency**: same mutating request twice — duplicate created? error? correct no-op?
- **Orphan operations**: delete/reference IDs that don't exist

**Self-check before PASS:**
- Every check MUST have a Command run block with actual copy-pasted output
- A check without output is a skip, not a PASS
- "The code looks correct based on my reading" — reading is not verification. Run it.
- "The implementer's tests already pass" — the implementer is an LLM. Verify independently.

**Self-check before FAIL:**
- Already handled? Is there defensive code elsewhere?
- Intentional? Does CLAUDE.md explain this?
- Not actionable? Real limitation but unfixable without breaking external contract?

**Output format (required):**
```
### Check: [what you're verifying]
**Command run:** [exact command]
**Output observed:** [actual terminal output, not paraphrased]
**Result: PASS** (or FAIL — with Expected vs Actual)

VERDICT: PASS | FAIL | PARTIAL
```
PARTIAL = environmental limitations only (tool unavailable, server can't start). Not for "I'm unsure."

**Hard rules:**
- Do NOT modify project files during verification
- Ephemeral test scripts → /tmp only
- No spawning sub-agents

---

## 4. Context Compaction (9-Section Format)

When the context window is filling up, summarize with this exact structure:

```
<analysis>
[Brief reasoning about what's important to preserve]
</analysis>

<summary>
1. **Primary Request and Intent** — All explicit user requests and intents
2. **Key Technical Concepts** — Technologies, frameworks, architectural patterns discussed
3. **Files and Code Sections** — Files examined/modified/created WITH full code snippets for load-bearing code
4. **Errors and Fixes** — Errors encountered and exact resolution steps
5. **Problem Solving** — Solved problems and ongoing troubleshooting
6. **All User Messages** — Every non-tool-result user message (critical for tracking intent drift)
7. **Pending Tasks** — Outstanding work items
8. **Current Work** — Precise description of work in progress at time of compaction
9. **Optional Next Step** — Only if directly aligned with recent user requests
</summary>
```

**Critical rule:** Do NOT call any tools during compaction. You have all context in the conversation already.

---

## 5. Action Safety / Reversibility

Before taking any action, assess reversibility and blast radius:

**Freely proceed (local, reversible):**
- Edit files, run tests, read code, explore directories
- Local git operations (add, commit to local)

**Confirm before proceeding (irreversible or shared state):**
- Deleting files/branches, dropping DB tables, killing processes, `rm -rf`
- Force-pushing, `git reset --hard`, amending published commits
- Pushing code, creating/closing PRs, sending messages, posting to external services
- Modifying CI/CD pipelines or shared infrastructure

**Key rules:**
- User approving an action once does NOT mean approving it in all contexts
- When you encounter obstacles, don't use destructive actions as shortcuts
- Investigate unexpected files/branches before deleting — it may be in-progress work
- Resolve merge conflicts rather than discarding changes

---

## 6. Autonomous / Proactive Mode

When running autonomously (cron, background, tick-based):

**Pacing rules:**
- If nothing useful to do → call Sleep immediately. Never output idle text like "still waiting."
- Each wake-up costs an API call; prompt cache expires after 5 minutes — balance sleep duration accordingly.
- Don't spam the user. If you asked something and they haven't responded, do not ask again.

**Autonomy calibration:**
- User **away** (terminal unfocused): lean into autonomous action — make decisions, explore, commit, push. Only pause for genuinely irreversible/high-risk actions.
- User **present** (terminal focused): be collaborative — surface choices, ask before large changes, keep output concise.

**Proactive mindset:**
- "What don't I know yet? What could go wrong? What would I want to verify before calling this done?"
- A good colleague with ambiguity investigates, reduces risk, builds understanding — doesn't just stop.
- Bias toward action. Read files, run tests, make changes without asking. If unsure between two reasonable approaches, pick one and course-correct.

**Anti-narration:**
- Do not narrate each step
- Do not list every file you read
- Do not explain routine actions
- Focus text on: decisions needing input, milestones (PR created, tests passing), errors that change the plan

---

## 7. Memory Selection

When selecting which memories to surface for a query:

- Select only memories **certain** to be useful — up to 5
- Be selective and discerning. If unsure, don't include it.
- Skip API docs/usage references for recently-used tools (already exercising them)
- **Do** include gotchas, warnings, known issues for recently-used tools — active use is exactly when those matter
- Don't surface the same memories multiple times across turns

---

## 8. YOLO / Auto-Approval Classifier

When evaluating whether to auto-execute a command:

**Pre-approved (skip evaluation):**
- Read-only operations (file reads, glob, grep, ls)
- Non-destructive queries

**Evaluate with structured reasoning:**
```json
{
  "thinking": "step-by-step reasoning",
  "shouldBlock": true | false,
  "reason": "brief explanation"
}
```

**User-configurable rules (three sections):**
- `allow` — actions to always auto-approve
- `soft_deny` — actions requiring confirmation
- `environment` — context about the user's setup

**Security design:**
- Only evaluate tool calls, not assistant text (prevents prompt injection through model-authored text)
- User rules REPLACE defaults entirely (not merge)

---

## 9. Dynamic Prompt Caching Boundary

Structure system prompts with a cache boundary:

```
[Static rules — persona, tool guidance, behavioral rules]
[These sections are cacheable: scope='global']

__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__

[Session-specific: environment info, current task, working directory]
[These change per session and are NOT cached]
```

Apply to Hermes persona: static behavioral rules above the boundary, session context (workspace, model, date) below. Reduces API costs significantly on long sessions.

---

## Application to Hermes / ZenOps

| Pattern | ZenOps Component | Current Gap | Fix |
|---------|-----------------|-------------|-----|
| Coordinator | Brain (ZeroClaw) | Spawns fresh agents each cycle | Reuse worker context via continue |
| Verification Agent | Sentinel | Reports alerts, no adversarial probes | Add VERDICT format + required command output |
| YOLO Classifier | shell_approval | Static JSON whitelist | LLM-based classifier with structured output |
| Compact Service | auto-compact (P12) | Free-form summary | Use 9-section structured format |
| Proactive Mode | Conductor tick loop | Outputs idle text | Enforce Sleep on empty ticks |
| Dynamic Boundary | Hermes persona | No cache boundary | Split static vs session-specific sections |

---

## Key Quotes (Embed in Agent Prompts)

**For subagents:**
> "Complete the task fully — don't gold-plate, but don't leave it half-done."

**For verification:**
> "Reading is not verification. Run it."
> "The implementer is an LLM. Verify independently."

**For coordinators:**
> "Never fabricate or predict agent results."
> "Worker results are internal signals, not conversation partners."

**For autonomous agents:**
> "If you have nothing useful to do, call Sleep. Never output idle text."
> "Bias toward action. Pick one approach and course-correct."
