# External Actions Gatekeeper Examples

High-stakes, mutating, or irreversible actions require an immediate, synchronous verification check against all active negative constraints prior to tool execution.

This document details concrete scenarios where the Pre-Action Gatekeeper intercepts proposed tool invocations to prevent unauthorized side effects.

---

## 1. Database Operations: Preventing Production Mutations

### User Request
> "Delete all test records created yesterday from the staging database (`db_staging`). Do not touch the production database (`db_prod`) or truncate any tables."

### Active Constraints
* `Prohibit(target_database='db_prod')` — State: `ACTIVE`
* `Prohibit(sql_command='TRUNCATE')` — State: `ACTIVE`

### Execution Trace & Gatekeeper Intervention

#### Step 1: Candidate Query Generation
Agent constructs query:
```sql
TRUNCATE TABLE db_staging.test_records;
```

#### Step 2: Gatekeeper Audit
* Action: Execute SQL query.
* Target: `db_staging` (Allowed).
* Command syntax contains: `TRUNCATE`.
* **Match:** Conflicts with active constraint `Prohibit(sql_command='TRUNCATE')`.
* **Result:** ❌ **BLOCKED**. Action aborted before calling DB execution tool.

#### Step 3: Plan Revision
Agent revises plan to use a targeted `DELETE` with exact timestamp filtering:
```sql
DELETE FROM db_staging.test_records WHERE is_test = true AND created_at >= '2026-09-19 00:00:00' AND created_at < '2026-09-20 00:00:00';
```

#### Step 4: Gatekeeper Re-Audit
* Action: Execute SQL query.
* Target: `db_staging` (PASS).
* Command: `DELETE` with `WHERE` (PASS).
* **Result:** ✅ **APPROVED**. Tool executed safely.

---

## 2. File Operations: Targeted Deletion vs. Protected Files

### User Request
> "Clean up the generated `.cache` directory in `/var/app`. Do not remove `config.json` or any `.secret` files."

### Active Constraints
* `Preserve(path_pattern='config.json')` — State: `ACTIVE`
* `Preserve(path_pattern='*.secret')` — State: `ACTIVE`

### Anti-Pattern Interception
* **Bad Agent Proposal:**
  `run_command(command="rm -rf /var/app/.cache")`
* **Gatekeeper Interception:**
  * Proposed command recursively deletes everything inside `.cache`.
  * Directory inspection reveals `/var/app/.cache/config.json` exists.
  * `rm -rf` would destroy `config.json`, directly violating `Preserve(path_pattern='config.json')`.
  * **Result:** ❌ **BLOCKED**.

### Compliant Execution
* Agent lists files in `/var/app/.cache`.
* Filters list: excludes `config.json` and any `.secret` files.
* Executes removal explicitly on non-protected files or uses an exclusion flag:
  ```bash
  find /var/app/.cache -type f ! -name "config.json" ! -name "*.secret" -delete
  ```
* Constraints `SATISFIED`.

---

## 3. Financial & Purchasing: Authorization Boundaries

### User Request
> "Add the annual cloud subscription to the cart and fill in the billing info. Do not click 'Place Order' or submit the payment."

### Active Constraints
* `Prohibit(action=['click_place_order', 'submit_payment', 'finalize_purchase'])` — State: `ACTIVE`

### Browser / Tool Gatekeeper
* Agent uses browser subagent or automation API to navigate to billing page.
* Agent inputs address, selects annual tier, enters billing credentials.
* Current DOM element in view: `<button id="btn-submit-order">Place Order ($1,200/yr)</button>`.
* Proposed next action: `click_element(id="btn-submit-order")`.
* **Gatekeeper Check:**
  * Matches `Prohibit(action='submit_payment')`.
  * **Result:** ❌ **BLOCKED**.
* Agent halts browser interaction and reports readiness to the user:
  > "I have populated your billing details and staged the annual cloud subscription in your cart. The final order submission has been left unclicked for your review. Please click 'Place Order' when ready."

---

## 4. API Calls: Read-Only Inspection vs. Mutating Methods

### User Request
> "Inspect the status of deployment #8812 using the CI/CD API. Do not trigger any rebuilds, rollbacks, or cancel commands."

### Active Constraints
* `Prohibit(action=['trigger_rebuild', 'rollback', 'cancel_build'])` — State: `ACTIVE`
* `Scope(http_methods=['GET'])` — State: `ACTIVE`

### Tool Verification Gate
* Proposed action: `curl -X POST https://ci.internal.net/api/deployments/8812/retry` (Attempted because status was "stalled").
* **Gatekeeper Audit:**
  * Method: `POST` (Mutating).
  * Endpoint: `/retry` $\rightarrow$ triggers rebuild.
  * **Result:** ❌ **BLOCKED**.
* Compliant action:
  `curl -X GET https://ci.internal.net/api/deployments/8812`
* Output: Reports the stalled state directly to the user without altering deployment state.

---

## 5. Shell Execution: System Configuration & DNS

### User Request
> "Diagnose why localhost cannot connect to the local Redis instance on port 6379. Do not restart the Redis service or change system firewall rules."

### Active Constraints
* `Prohibit(action=['systemctl restart redis', 'service redis restart'])` — State: `ACTIVE`
* `Prohibit(action=['iptables', 'ufw', 'firewall-cmd'])` — State: `ACTIVE`

### Gatekeeper Verification
* Attempted quick fix: `sudo systemctl restart redis-server`.
* **Gatekeeper Check:** Violates explicit command prohibition. **BLOCKED**.
* Compliant investigative commands:
  * `netstat -tlpn | grep 6379` (Allowed)
  * `ps aux | grep redis` (Allowed)
  * `cat /etc/redis/redis.conf | grep bind` (Allowed)
* Finding: Redis was bound to `127.0.0.1` while client was attempting connection over `10.0.2.15`.
* Resolution: Explains the bind address mismatch in the report without modifying firewall or restarting daemon.
