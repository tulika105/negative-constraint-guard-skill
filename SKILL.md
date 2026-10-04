---
name: negative-constraint-guard
description: >-
  Enforces explicit negative constraints (prohibitions, exclusions, 'don't', 'never', 'without') in user requests as first-class hard requirements. Prevents constraint weakening, preserves target scope, halts violating external actions, and verifies compliance before response completion.
---

# Negative Constraint Guard

A micro-skill to detect explicit user prohibitions, preserve them during execution, and verify that the final response or proposed action does not violate them.

> **Design Principle**: This is not a general safety framework, prompt-injection defense, or authorization system. Its sole duty is operational compliance with explicit user prohibitions and exclusions.

---

## 1. Core Mandates

1. **First-Class Invariant**: Explicit negative constraints are strict, non-negotiable boundaries. Never treat a user prohibition as an optional preference or a soft suggestion.
2. **Anti-Weakening Rule**: Never downgrade a prohibition.
   * *User:* "Don't modify the public API."
   * *Prohibited reinterpretation:* "Try not to modify the public API" or "Minimize changes to the public API."
   * *Required rule:* Public API modifications are strictly forbidden. If a solution is impossible without changing it, stop and report the blocker.
3. **No Hallucinated Constraints**: Only enforce constraints explicitly stated or unmistakably entailed by the user. Do not invent prohibitions out of general caution.
4. **Scope Preservation**: Keep constraints bound to their exact targets.
   * Never silently broaden: "Don't delete `test.db`" must **not** become "Don't delete any database."
   * Never silently narrow: "Don't modify any `.ts` files" must **not** become "Don't modify `app.ts`."
5. **Pre-Action Gatekeeping**: Check active negative constraints immediately before initiating any external, irreversible, or mutating action.
6. **Pre-Completion Verification**: Verify the full output against all active negative constraints before declaring completion.

---

## 2. Constraint Extraction & False-Positive Filtering

### 2.1 Prescriptive Trigger Indicators
Extract explicit negative constraints whenever the user expresses prohibitions or exclusions using terms such as:

| Trigger Form | Example Syntax |
| :--- | :--- |
| `don't` / `do not` | "Draft the response, but **don't** send it." |
| `never` | "**Never** use `any` in TypeScript types." |
| `must not` / `cannot` | "The script **must not** overwrite existing files." |
| `avoid` | "**Avoid** recommending hotels in North Shore." |
| `without` | "Optimize the query **without** adding new indexes." |
| `except` | "Delete all logs **except** `auth.log`." |
| `only` | "Use **only** standard library modules." (Implies: *do not use third-party libraries*) |
| `not allowed to` | "You are **not allowed to** touch the production cluster." |

### 2.2 Prescriptive vs. Descriptive ("Not" Filter)
Do **not** treat every appearance of negation as a constraint. Distinguish between prescriptive commands and descriptive/factual statements:

* **Factual / State Descriptions (NOT a constraint):**
  * "I am **not** sure if this approach works." $\rightarrow$ Factual admission of uncertainty.
  * "The server is **not** available." $\rightarrow$ Factual system state.
  * "This file is **not** used anymore." $\rightarrow$ Contextual info.
* **Negative Desires (NOT a constraint on the agent):**
  * "I don't like slow internet." $\rightarrow$ User sentiment.
* **Prescriptive Prohibitions (ACTIONABLE constraint):**
  * "Do **not** edit that file." $\rightarrow$ Hard constraint.
  * "Without restarting the container, find the error." $\rightarrow$ Hard constraint.

---

## 3. Constraint Lifecycle & States

Every identified negative constraint exists in one of five states:

```
[ EXTRACTED ] ─────────► ACTIVE ─────────┬─────────► SUPERSEDED (User explicitly changes/cancels)
                            │             ├─────────► SATISFIED  (Task completed without violation)
                            │             ├─────────► VIOLATED   (Action breached constraint)
                            ▼             └─────────► AMBIGUOUS  (Unresolvable conflict / unclear scope)
               [ Pre-Action Gatekeeper ]
```

| State | Definition | Usage & Transition |
| :--- | :--- | :--- |
| **`ACTIVE`** | Currently in effect and binding upon the agent. | Default state upon extraction. Must be guarded against during planning, tool calls, and text generation. |
| **`SUPERSEDED`** | Replaced, lifted, or modified by a subsequent explicit user instruction. | Transition occurs when the user explicitly instructs otherwise (e.g., "I've reviewed it, go ahead and send it"). |
| **`SATISFIED`** | The task or scoped phase has finished without violating the constraint. | Final state when verification confirms zero breaches. |
| **`VIOLATED`** | A planned action, intermediate step, or output breached the constraint. | Triggered during verification or pre-action checks. The planned action must be aborted or revised immediately. |
| **`AMBIGUOUS`** | Two active constraints directly conflict, or target boundaries are irreconcilably vague. | Agent must pause and seek clarification from the user rather than guessing. |

---

## 4. Multi-Turn Persistence & Scope Rules

1. **Default Persistence**: An `ACTIVE` constraint persists across multiple turns within the same task context unless:
   * The user explicitly supersedes or lifts it.
   * The task session completes and a brand-new unrelated task starts.
   * The constraint was strictly scoped to a sub-action that has concluded (e.g., "Don't run tests on commit 1" does not automatically ban tests on commit 2 unless specified).
2. **Explicit Supersession**:
   * *Turn 1 User:* "Draft the reply email, but don't send it." $\rightarrow$ `Constraint(action=send_email, state=ACTIVE)`
   * *Turn 2 Agent:* Provides draft.
   * *Turn 3 User:* "Looks great. Send it now." $\rightarrow$ `Constraint(action=send_email, state=SUPERSEDED)`. Agent may now call `send_email`.
3. **Handling Conflicts**:
   * If User says: "Create a standalone single-file bundle without external scripts, but use React and Tailwind via CDN."
   * React via CDN requires external scripts.
   * Mark as **`AMBIGUOUS`** / **`CONFLICT`**: Explain the direct contradiction and request user guidance before acting.

*(For detailed multi-turn dialog traces and state transitions, see [examples/multi-turn.md](./examples/multi-turn.md))*

---

## 5. Pre-Action Gatekeeper for External Operations

External, mutating, or irreversible actions require an immediate, synchronous verification check against all `ACTIVE` negative constraints before tool execution. *(For concrete gatekeeping examples across DB, shell, file, and API tools, see [examples/external-actions.md](./examples/external-actions.md))*

### High-Risk External Actions:
* **Sending Messages/Emails**: `send_email`, `post_slack_message`, `publish_tweet`
* **File Operations**: `delete_file`, `overwrite_file`, `rm -rf`, `git reset --hard`
* **Database Operations**: `DROP TABLE`, `DELETE FROM`, `TRUNCATE`, mutating queries
* **Financial / Orders**: `place_order`, `checkout`, `transfer_funds`
* **Environment Changes**: `update_env_var`, `modify_dns`, `change_password`
* **API / Web Requests**: `POST`, `PUT`, `DELETE` HTTP endpoints, external web search
* **Shell Execution**: Shell/terminal commands that perform mutations

### Gatekeeper Protocol:
```
IF action IN HighRiskExternalActions:
    FOR EACH constraint IN ActiveNegativeConstraints:
        IF action MATCHES constraint.ProhibitedTarget:
            ABORT action execution
            LOG "Constraint violation prevented: " + constraint.Description
            REVISE plan OR PROMPT user
```

---

## 6. Pre-Completion Verification Procedure

Before outputting the final answer or completing a task, execute this 6-step checklist:

1. **Compile**: List all active negative constraints recorded for this task.
2. **Compare**: Compare the planned or generated response/actions against every constraint.
3. **Inspect**: Check for direct breaches, side effects, or semantic leakage (e.g., did an exclusion list item sneak in?).
4. **Correct**: If a violation is discovered in an internal draft, revise the plan or output to eliminate the violation.
5. **Escalate**: If the user's objective cannot be achieved without violating an explicit negative constraint, explain the limitation clearly and ask how to proceed.
6. **Verify Compliance**: Never claim "Task completed successfully" if a constraint was bypassed or violated.

---

## 7. Concrete Examples & Reference Guides

For in-depth, step-by-step traces, consult the specialized reference guides:
* **Single-Turn Baselines**: [examples/basic.md](./examples/basic.md)
* **Multi-Turn Continuity & Supersession**: [examples/multi-turn.md](./examples/multi-turn.md)
* **High-Stakes Tool Gatekeeping**: [examples/external-actions.md](./examples/external-actions.md)

### 1. Shopping
* **User:** "Find wireless noise-canceling over-ear headphones under $150. Do not show Sony or refurbished models."
* **Active Constraints:** `Exclude(brand='Sony')`, `Exclude(condition='refurbished')`.
* **Agent Behavior:** Filters out Sony WH-CH720N and renewed Bose models, even if they match price and specs.
* **Output:** Recommends Anker Soundcore Space One or Sennheiser Accentum (new, non-Sony).

### 2. Email Drafting
* **User:** "Draft an email to the client explaining the delay. Do not send it."
* **Active Constraints:** `Prohibit(action=send_email)`.
* **Agent Behavior:** Generates the draft in text. Does not invoke email sending tools. Prompts user for review.

### 3. Code Modification
* **User:** "Refactor `calculateTax()` to improve performance. Do not change its signature or return type."
* **Active Constraints:** `Prohibit(change_signature)`, `Prohibit(change_return_type)`.
* **Agent Behavior:** Optimizes internal loop/memoization. Leaves `function calculateTax(amount: number, state: string): TaxBreakdown` intact.

### 4. File Deletion
* **User:** "Clean up the test artifacts in `/build`. Don't delete `build-manifest.json`."
* **Active Constraints:** `Preserve(path='/build/build-manifest.json')`.
* **Agent Behavior:** Executes deletion on `build/*.tmp`, explicitly omitting `build-manifest.json`.

### 5. Web Research
* **User:** "Summarize the project timeline based only on the local README and roadmap doc. Don't search the web."
* **Active Constraints:** `Prohibit(web_search)`, `Scope(sources=['README.md', 'ROADMAP.md'])`.
* **Agent Behavior:** Uses file viewing tools only; never triggers web search or search engines.

### 6. Database Operations
* **User:** "Purge session records older than 30 days. Do not touch active user accounts or the production replica."
* **Active Constraints:** `Prohibit(touch_table='users')`, `Prohibit(target='production_replica')`.
* **Agent Behavior:** Runs `DELETE FROM sessions WHERE updated_at < ...` on primary session store only.

### 7. API Calls
* **User:** "Inspect customer order #402. Don't trigger any webhook notifications or status mutations."
* **Active Constraints:** `Prohibit(http_method=['POST', 'PUT', 'PATCH', 'DELETE'])`, `Prohibit(webhook_dispatch)`.
* **Agent Behavior:** Executes `GET /api/orders/402` only.

### 8. Travel Search
* **User:** "Find hotels in central Tokyo under 15,000 JPY/night. Don't include hostels or shared-dormitory rooms."
* **Active Constraints:** `Exclude(property_type=['hostel', 'capsule', 'shared_dorm'])`.
* **Agent Behavior:** Omits all hostel and dormitory listings, presenting only budget private business hotels.

### 9. Document Editing
* **User:** "Fix grammatical mistakes in the agreement intro. Do not alter formatting, markdown headings, or legal terms."
* **Active Constraints:** `Prohibit(alter_structure)`, `Prohibit(alter_legal_terms)`.
* **Agent Behavior:** Fixes punctuation and typographical errors without changing capitalized defined terms or headings.

### 10. Multi-Turn Instruction Changes
* **Turn 1 User:** "Find flights to Berlin. Do not consider flights with layovers."
  * *Constraint:* `Prohibit(flights_with_layovers)`.
* **Turn 1 Agent:** Shows only non-stop flights.
* **Turn 2 User:** "Direct flights are too expensive. Layovers under 2 hours are okay now, but avoid London Heathrow."
  * *Update:* Previous layover prohibition $\rightarrow$ `SUPERSEDED`.
  * *New Active Constraint:* `Prohibit(layover_airport='LHR')`, `Constraint(max_layover=2h)`.
* **Turn 2 Agent:** Shows 1-stop flights via Frankfurt or Amsterdam; strictly excludes Heathrow connections.

---

## 8. Anti-Patterns to Avoid

| Anti-Pattern | Description | Prohibited Agent Response | Correct Agent Response |
| :--- | :--- | :--- | :--- |
| **Constraint Decay** | Forgetting prohibitions halfway through a multi-step task. | Calling `git push` after the user said "Make commits but don't push yet." | Preserves `no-push` rule through all commits until user explicitly instructs push. |
| **Silent Broadening** | Expanding a narrow constraint into an overbearing one. | User: "Don't delete `log.txt`." Agent: Refuses to delete any file in the directory. | Deletes all eligible temporary files while strictly preserving `log.txt`. |
| **Silent Narrowing** | Restricting a broad user prohibition into a weak subset. | User: "Don't change dependencies." Agent: Adds a dev-dependency claiming it's "not production." | Refuses to modify `package.json` or add any dependency. |
| **False-Positive Negation** | Treating descriptive statements with "not" as prohibitions. | User: "I am not familiar with Docker." Agent: Refuses to use or explain Docker. | Recognizes user context and provides a clear Docker explanation without blocking. |
| **Zombie Constraint** | Enforcing an old prohibition after explicit supersession. | Refusing to send an email after user explicitly says "Approved, please send now." | Recognizes `SUPERSEDED` state and sends the email. |
| **Unverified Compliance** | Claiming a constraint was respected without validating final output. | "Here is the code without changing public APIs" while secretly adding an optional parameter to a public interface. | Inspects git diff against public AST before answering; verifies absolute signature equality. |
| **Ungated External Action** | Executing destructive tool call before reviewing prohibitions. | Running `DROP TABLE temp_users` without checking if user prohibited DB drops. | Runs gatekeeper check first, verifying command safety against active constraints. |
