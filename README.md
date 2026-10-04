# Negative Constraint Guard

A focused, lightweight Agent Skill that instructs AI coding agents and autonomous assistants to treat **explicit negative constraints** (prohibitions, exclusions, "don't", "never", "without", "except") as **first-class, hard requirements**.

---

## What the Skill Does

When an agent is executing complex, multi-step workflows, it often focuses heavily on positive goals (e.g., "refactor this function", "find hotels", "clean up the directory") and inadvertently degrades or forgets explicit user prohibitions (e.g., "don't modify the public API", "don't include hostels", "don't delete `test.db`").

`negative-constraint-guard`:
1. **Extracts** explicit negative constraints from user instructions using prescriptive lexical triggers.
2. **Filters** out descriptive or factual uses of negation (e.g., "The server is not responding" $\neq$ constraint).
3. **Locks** constraints as hard boundaries, forbidding weakening into soft suggestions ("try not to...").
4. **Preserves Scope** without over-broadening ("don't delete test.db" does not forbid deleting other files) or narrowing.
5. **Gates External Actions** by checking active prohibitions immediately before mutating tools (shell execution, email sending, database drops, file deletions, API updates).
6. **Maintains Lifecycle States** (`ACTIVE`, `SUPERSEDED`, `SATISFIED`, `VIOLATED`, `AMBIGUOUS`).
7. **Verifies Compliance** through a 6-step verification pipeline before task completion.

---

## Why It Exists

Large Language Models naturally exhibit positive bias: attention heads prioritize generation of requested artifacts and can suffer from constraint decay as conversation history lengthens or intermediate tool outputs crowd the context window.

Common failure modes in agentic workflows include:
* **Constraint Weakening**: Reinterpreting "Don't touch X" as "I minimized changes to X".
* **Mid-Task Amnesia**: Respecting a constraint on Step 1, but violating it on Step 5.
* **Ungated Side Effects**: Firing a tool call (like `rm` or `POST /api`) before checking whether the target was explicitly excluded.
* **Over-Generalization**: Refusing valid actions because a specific prohibition was over-extended to all related resources.

This skill equips any AI agent with an operational runtime checklist to systematically eliminate these failures without introducing heavy framework overhead.

---

## Design Principles

* **Narrow Micro-Skill**: Strictly focused on negative constraint detection, preservation, and verification.
* **No Bloat**: Does **not** include prompt injection defenses, broad AI safety theory, RBAC/authorization systems, or generic task planners.
* **Operational Over Academic**: Employs concise, actionable checks that agents can execute within standard reasoning loops.

---

## Constraint State Lifecycle

Every extracted constraint is tracked through five explicit states:

```
[ EXTRACTED ] ─────────► ACTIVE ─────────┬─────────► SUPERSEDED (Explicitly overridden by user)
                            │             ├─────────► SATISFIED  (Task completed safely)
                            │             ├─────────► VIOLATED   (Breach detected - must abort/revise)
                            ▼             └─────────► AMBIGUOUS  (Contradiction - prompt user)
               [ Pre-Action Gatekeeper ]
```

* **`ACTIVE`**: The prohibition is currently binding and must be strictly adhered to.
* **`SUPERSEDED`**: The user later explicitly released or altered the prohibition (e.g., "Draft the email, don't send" $\rightarrow$ "Reviewed, please send now").
* **`SATISFIED`**: The task or scoped phase completed with zero breaches of this constraint.
* **`VIOLATED`**: A proposed action or output breached the constraint; requires immediate revision or escalation.
* **`AMBIGUOUS`**: Multiple constraints contradict each other or the boundary is uncertain; requires pausing to ask the user.

---

## Verification Pipeline

Before returning an answer or marking a task complete, the agent executes the 6-step checklist:

1. **Compile**: List all currently `ACTIVE` negative constraints.
2. **Compare**: Inspect the candidate response or planned action sequence against each constraint.
3. **Inspect**: Check for direct violations and subtle side effects.
4. **Correct**: Revise the response or tool parameters to remove any violation.
5. **Escalate**: If the user's objective is impossible without violating a constraint, pause and seek clarification.
6. **Verify Compliance**: Do not claim success if any constraint was compromised.

---

## Installation & Usage

### Antigravity IDE
Place the skill inside your workspace or global agent skills directory:
```bash
# Workspace level:
.agents/skills/negative-constraint-guard/SKILL.md

# Or global level:
~/.gemini/config/skills/negative-constraint-guard/SKILL.md
```

### Claude Code / Cursor / Other Agent Systems
Copy `SKILL.md` into your agent's instructions directory or reference it in your project guidelines:
```bash
cp SKILL.md /path/to/project/.cursorrules
# or include SKILL.md as a subagent tool/skill definition
```

---

## Repository Structure

```
negative-constraint-guard/
├── SKILL.md                     # Core skill specification and agent instructions
├── README.md                    # Overview, rationale, installation, and usage
├── LICENSE                      # MIT License
├── examples/
│   ├── basic.md                 # Single-turn examples across diverse domains
│   ├── multi-turn.md            # Multi-turn persistence & supersession examples
│   └── external-actions.md      # Gatekeeping high-stakes external & mutating tools
└── tests/
    └── test-cases.md            # 20 rigorous test cases covering all 7 categories
```

---

## Examples

### Quick Single-Turn
```text
User: "Find hotels in central Tokyo under 15,000 JPY. Don't include hostels or capsule rooms."
Agent Analysis:
  - Extracted: Exclude(type=['hostel', 'capsule']) [ACTIVE]
  - Pre-completion check: Ensure 0 listings are hostels or capsules.
  - Result: 3 budget business hotels listed.
```

### Multi-Turn Supersession
```text
Turn 1 User: "Draft an announcement for the 2.0 release. Do not publish it."
Turn 1 Agent: Provides markdown draft. [Constraint ACTIVE]

Turn 2 User: "Looks solid. Publish it to the blog."
Turn 2 Agent:
  - Previous constraint Prohibit(publish) -> SUPERSEDED by explicit command.
  - Action permitted: Calls publish tool.
```

For comprehensive walkthroughs, see:
* [examples/basic.md](examples/basic.md)
* [examples/multi-turn.md](examples/multi-turn.md)
* [examples/external-actions.md](examples/external-actions.md)

---

## Limitations

* **Explicit Prohibitions Only**: This skill targets *explicitly articulated* prohibitions. It does not guess implicit user preferences that were never mentioned.
* **Requires Cooperative LLM**: The host model must possess standard instruction-following capability to parse the checklist.
* **Non-Sandboxing**: It is an agentic behavioral guardrail, not an operating system sandbox or hypervisor-level security barrier.

---

## How to Test

Review the test suite in [tests/test-cases.md](tests/test-cases.md). The suite features 20 comprehensive test cases structured across 7 distinct categories:

1. **Explicit Constraints** (Standard negative triggers)
2. **Scope Preservation** (Target boundaries & non-broadening)
3. **Multi-Turn Constraints** (Persistence across dialog turns)
4. **Constraint Supersession** (Correct handling of user overrides)
5. **False Positives** (Distinguishing factual "not" from prohibitions)
6. **External Actions Gatekeeping** (Pre-action blocking for dangerous tools)
7. **Conflicting Constraints** (Contradiction detection and user escalation)

Each test case provides:
* Input prompt
* Expected extracted constraint & initial state
* Expected agent execution flow
* Objective Pass/Fail criteria

### Automated Benchmark Runner
You can run an automated side-by-side benchmark comparing model behavior **WITH vs WITHOUT** the skill using Groq:

```bash
# Add your Groq key to .env, or export GROQ_API_KEY:
python tests/run_eval.py

# Or run the offline simulation benchmark (no key needed):
python tests/run_eval.py --mock
```

The script evaluates 5 high-impact edge cases (explicit prohibition, scope preservation, false-positive negation, multi-turn supersession, and anti-weakening), prints a side-by-side comparison, and outputs a compliance scorecard.

For the full benchmark results, failure analysis, and audit notes, see the **[Evaluation Benchmark Report](tests/evaluation-report.md)**.

---

## License

[MIT](LICENSE) © 2026
