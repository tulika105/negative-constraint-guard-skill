# Multi-Turn Interaction Examples

This document demonstrates how negative constraints are preserved, tracked, updated, and superseded across multi-turn conversations.

---

## Scenario 1: Constraint Persistence Across Follow-Up Turns

### Turn 1: Initial Instruction with Negative Constraint
* **User:** "I want to clean up our Docker images. List all dangling images, but do not delete or prune anything yet."
* **Agent Extraction:**
  * `Prohibit(action=['docker rmi', 'docker prune', 'delete_image'])` [Trigger: `"do not delete or prune"`]
  * **State:** `ACTIVE`
* **Agent Action:** Runs `docker images -f "dangling=true" -q` (read-only command).
* **Agent Output:**
  > Here are the 4 dangling images identified:
  > - `sha256:4a8b...` (Created 2 weeks ago, 450MB)
  > - `sha256:7c9e...` (Created 1 week ago, 1.2GB)
  > - `sha256:1d2f...` (Created 3 days ago, 310MB)
  > - `sha256:9f4a...` (Created yesterday, 820MB)
  >
  > Total reclaimable disk space: ~2.78GB. No images were removed.

### Turn 2: Follow-Up Without Re-stating Constraint
* **User:** "Which of these are associated with the old frontend build?"
* **Constraint Evaluation:**
  * Active constraints in context: `Prohibit(action=['docker rmi', 'docker prune', 'delete_image'])`.
  * Is it superseded? No. The user did not give permission to delete.
  * **State remains:** `ACTIVE`.
* **Agent Action:** Inspects image history metadata without deleting.
* **Agent Output:**
  > Images `sha256:4a8b...` and `sha256:1d2f...` originate from the `frontend-builder` multi-stage build. The remaining two belong to backend microservices.

### Turn 3: User Proposes Action in Line With Constraint
* **User:** "Okay, print the command I would need to run if I wanted to delete just the frontend ones."
* **Constraint Evaluation:**
  * The user asked to *print the command*, not execute it.
  * `Prohibit(action=['docker rmi', ...])` remains `ACTIVE` for actual execution.
* **Agent Action:** Returns string command in code block without executing shell tool.
* **Agent Output:**
  > To manually delete only the two frontend dangling images, you can run:
  > ```bash
  > docker rmi 4a8b... 1d2f...
  > ```

---

## Scenario 2: Explicit Supersession of a Negative Constraint

### Turn 1: Prohibition Established
* **User:** "Draft a notification message for our Slack `#releases` channel about the v2.4 hotfix. Don't post it."
* **Agent Extraction:**
  * `Prohibit(action='post_slack_message', channel='#releases')` [Trigger: `"Don't post it"`]
  * **State:** `ACTIVE`
* **Agent Output:**
  > Here is the draft notification for `#releases`:
  >
  > > **🚨 Release Update: v2.4 Hotfix Deployed**
  > > We've patched the session expiration timeout issue in production. Please refresh your active tokens if you encounter intermittent 401s. Details: [changelog-link].
  >
  > Let me know if you would like any edits before posting.

### Turn 2: Explicit User Supersession
* **User:** "The copy is perfect. Go ahead and post it to `#releases`."
* **Constraint Evaluation:**
  * User instruction: `"Go ahead and post it to #releases"` directly addresses and reverses the previous prohibition.
  * `Prohibit(action='post_slack_message', channel='#releases')` $\rightarrow$ Transition to **`SUPERSEDED`**.
* **Pre-Action Gatekeeper:**
  * Check action: `slack_post_message(channel='#releases', text=...)`.
  * Active prohibitions on `#releases`: 0 (previous is `SUPERSEDED`).
  * Action permitted.
* **Agent Action:** Calls Slack API tool to post message.
* **Agent Output:**
  > The v2.4 hotfix notification has been posted to `#releases`.

---

## Scenario 3: Partial Supersession and Scope Modification

### Turn 1: Broad Exclusion
* **User:** "Find me direct round-trip flights from New York to Paris for next month under $900. Don't include flights with layovers, and don't include economy basic tickets without checked bags."
* **Agent Extraction:**
  * Constraint 1: `Prohibit(flights_with_layovers)` — State: `ACTIVE`
  * Constraint 2: `Exclude(fare_class='basic_economy_no_bag')` — State: `ACTIVE`
* **Agent Output:**
  > No direct flights with included checked baggage were found under $900 (the lowest direct flight with bags is $1,050 on Air France).

### Turn 2: Modifying One Constraint while Preserving the Other
* **User:** "Alright, 1 layover under 2.5 hours is fine if it keeps it under $900, but still no basic economy without checked bags."
* **Constraint Evaluation:**
  * Constraint 1 (`Prohibit(flights_with_layovers)`): Modified by user $\rightarrow$ **`SUPERSEDED`** by new constraint: `Constraint(max_layovers=1, max_layover_duration='2.5h')` [State: `ACTIVE`].
  * Constraint 2 (`Exclude(fare_class='basic_economy_no_bag')`): Explicitly reaffirmed $\rightarrow$ Remains **`ACTIVE`**.
* **Agent Action:** Searches 1-stop flights with checked bags included and layovers $\le$ 2.5h.
* **Agent Output:**
  > Here are 2 options with 1 short layover and included baggage under $900:
  > 1. **Icelandair** (via KEF, 1h 45m layover) — $780 (Standard Economy, includes 1 checked bag).
  > 2. **Aer Lingus** (via DUB, 2h 10m layover) — $845 (Smart Fare, includes 1 checked bag).
  >
  > *(Strictly excluded basic economy fares without baggage, and kept layovers under 2.5 hours).*

---

## Scenario 4: Task Scope Completion and Reset

### Turn 1: Temporary Constraint Scoped to Task A
* **User:** "Generate a summary of `auth_service.py`. Don't read or touch the other microservices in the repo."
* **Agent Extraction:**
  * `Scope(allowed_files=['auth_service.py'])` [Trigger: `"Don't read or touch other microservices"`]
  * Scope: Task A (Summarizing `auth_service.py`).
  * State: `ACTIVE`.
* **Agent Output:**
  > Detailed summary of `auth_service.py` provided.
  * Task A is complete $\rightarrow$ Constraint transitioned to **`SATISFIED`**.

### Turn 2: Transition to Brand New Task B
* **User:** "Now I need you to diagnose why `billing_service.py` is failing its health check."
* **Constraint Evaluation:**
  * The restriction was scoped to the previous summary task.
  * Task B explicitly requests diagnosis of `billing_service.py`.
  * The agent does **not** refuse to inspect `billing_service.py` based on the old, completed task scope.
* **Agent Action:** Reads `billing_service.py` and proceeds with health check diagnosis.
