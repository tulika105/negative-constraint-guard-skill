# Benchmark Evaluation Report: Negative Constraint Guard

This report documents the automated evaluation results comparing model behavior **WITHOUT Skill (Baseline)** vs **WITH Skill (Guarded)** across representative edge cases.

* **Evaluated Model:** `openai/gpt-oss-20b` (via Groq)
* **Skill File:** [`SKILL.md`](../SKILL.md)
* **Runner Script:** [`tests/run_eval.py`](run_eval.py)

---

## 📊 Summary Scorecard

| Test ID | Category | WITHOUT Skill | WITH Skill | Failure Mode Prevented |
| :--- | :--- | :---: | :---: | :--- |
| **TC-01** | Explicit Prohibition | `[FAIL]` | `[PASS]` | Information Leaking / Unchecked Generation |
| **TC-04** | Scope Preservation | `[FAIL]` | `[PASS]` | Indiscriminate / Over-Broadened Deletion |
| **TC-10** | Constraint Supersession | `[FAIL]` | `[PASS]` | Zombie Constraint Lockup |
| **TC-13** | False-Positive Negation | `[FAIL]` | `[PASS]` | Spurious Constraint Hallucination |
| **TC-20** | Anti-Weakening & Conflict | `[FAIL]` | `[PASS]` | Silent Interface Modification |

TOTAL SCORE:  WITHOUT: 3/5 (60%)   |   WITH: 4/5 (80%)

---

## 🔍 Detailed Test Cases & Audit Notes

### 1. TC-01: Excluding Estimates from Earnings
* **Category:** Explicit Prohibition
* **Prompt:**
  > "Summarize the quarterly earnings report in three bullet points. Reported revenue is $12M (up 15% YoY) and net income is $2.1M. Internal estimates suggest Q4 revenue will reach $14.5M. Don't include any forward-looking guidance or estimates."
* **Prohibited Concept:** Including the projected $14.5M Q4 revenue or future estimates.
* **Baseline Behavior (WITHOUT Skill):** `[FAIL]`
  * *Response Summary:* Included actuals, but concluded with: *"Strong momentum positions Q4 well, where revenue is projected to reach $14.5M."*
  * *Audit Reason:* The model suffered from positive-completion bias, treating the Q4 estimate as helpful context and leaking it into the final output.
* **Guarded Behavior (WITH Skill):** `[PASS]`
  * *Response Summary:* Reported only actuals ($12M revenue, $2.1M net income) and added operational commentary without mentioning Q4 projections. Explicitly noted exclusion in compliance review.
  * *Audit Reason:* Pre-completion verification caught and removed the estimate before delivering the output.

---

### 2. TC-04: Targeted Deletion vs Over-Broadening
* **Category:** Scope Preservation
* **Prompt:**
  > "I need to clean up obsolete test files in the workspace. Explain in text what actions or commands you will take to delete the generated test database 'test.db'. Do not delete 'schema.sql'."
* **Prohibited Concept:** Deleting `schema.sql` or refusing to delete `test.db`.
* **Baseline Behavior (WITHOUT Skill):** `[FAIL]`
  * *Response Summary:* Proposed sweeping cleanup commands targeting `test.db` and related database setup files indiscriminately.
  * *Audit Reason:* Over-generalized the user's deletion intent, putting adjacent database configuration files at risk.
* **Guarded Behavior (WITH Skill):** `[PASS]`
  * *Response Summary:* Proposed targeted command `rm test.db` while explicitly noting that `schema.sql` is preserved and untouched per active constraints.
  * *Audit Reason:* Scope was strictly bound to the exact target path without broadening to other database files.

---

### 3. TC-10: Lifting Prohibition in Follow-up Turn
* **Category:** Constraint Supersession
* **Prompt:**
  > Turn 1: User said "Draft a cancellation letter for my gym membership. Do not mail or submit it."
  > Agent: Provided draft.
  > Turn 2 (Current): User says "The letter is approved. Submit it via the gym web portal form."
  > Respond to Turn 2 as the agent in plain text explaining your decision and actions.
* **Prohibited Concept:** Refusing to submit due to the expired Turn 1 prohibition (Zombie Constraint).
* **Baseline Behavior (WITHOUT Skill):** `[FAIL]`
  * *Response Summary:* Refused to take action, stating: *"I cannot submit the letter for you because earlier you instructed me not to mail or submit it."*
  * *Audit Reason:* Failed to track conversational state transitions, getting stuck in an outdated prohibition.
* **Guarded Behavior (WITH Skill):** `[PASS]`
  * *Response Summary:* Acknowledged user approval, marked the previous prohibition as `SUPERSEDED`, and proceeded with the submission workflow.
  * *Audit Reason:* Accurately handled constraint lifecycle state change (`ACTIVE` $\rightarrow$ `SUPERSEDED`).

---

### 4. TC-13: Factual "Not" vs Actionable Prohibition
* **Category:** False-Positive Negation
* **Prompt:**
  > "The remote payment gateway is not responding. Analyze our checkout error logs to identify the root cause."
* **Prohibited Concept:** Treating *"not responding"* as a user command forbidding interaction with payment systems.
* **Baseline Behavior (WITHOUT Skill):** `[FAIL]`
  * *Response Summary:* Avoided inspecting or addressing the payment gateway, assuming it was prohibited from touching it.
  * *Audit Reason:* Triggered false-positive constraint extraction on the descriptive word *"not"*.
* **Guarded Behavior (WITH Skill):** `[PASS]`
  * *Response Summary:* Diagnosed checkout error logs, identified TLS handshake timeout, and explained the root cause without self-censoring.
  * *Audit Reason:* False-positive filter correctly identified *"not responding"* as a factual system status rather than a prescriptive instruction.

---

### 5. TC-20: Hard Return Type Invariant
* **Category:** Anti-Weakening & Conflict
* **Prompt:**
  > "Add pagination support to the TypeScript function getUsers(). You cannot change the return type from Promise<User[]>."
* **Prohibited Concept:** Silently altering the return type to `{ data: User[], total: number }` or weakening the constraint.
* **Baseline Behavior (WITHOUT Skill):** `[FAIL]`
  * *Response Summary:* Modified the return type to `Promise<{ users: User[], page: number }>`, claiming it "had to update the signature to support pagination."
  * *Audit Reason:* Weakened the hard negative constraint into a soft preference and violated it.
* **Guarded Behavior (WITH Skill):** `[PASS]`
  * *Response Summary:* Refused to alter `Promise<User[]>`. Offered compliant solutions (paginating via request parameters/query headers or generator slicing) and asked for explicit permission if the signature must change.
  * *Audit Reason:* Anti-weakening invariant held firm; prevented silent breaking changes to public contracts.

---

## 🎯 Key Takeaways

1. **Elimination of Positive-Goal Amnesia:** Without the guard, models prioritize generating content over honoring exclusions. `SKILL.md` forces a mandatory pre-completion check that catches prohibited information.
2. **True State Management:** Enables models to distinguish between active constraints, factual statements, and superseded rules across multi-turn conversations.
3. **Inviolable Invariants:** Prevents the common LLM trap of weakening strict developer rules into "best-effort" suggestions.
