# Basic Single-Turn Examples

This document demonstrates single-turn interactions where an agent detects, isolates, and adheres to explicit negative constraints across standard day-to-day domains.

---

## 1. Shopping: Brand & Condition Exclusions

### User Prompt
> "Find me 3 mechanical gaming keyboards under $100. Do not include wireless models or refurbished items, and avoid Razer products."

### Constraint Analysis
* **Extracted Constraints:**
  1. `Exclude(connectivity='wireless')` [Trigger: `"Do not include"`] — State: `ACTIVE`
  2. `Exclude(condition='refurbished')` [Trigger: `"Do not include"`] — State: `ACTIVE`
  3. `Exclude(brand='Razer')` [Trigger: `"avoid"`] — State: `ACTIVE`
* **Target Scope:** Only the recommended keyboard items.
* **Positive Goal:** Recommend 3 wired mechanical gaming keyboards < $100.

### Pre-Completion Checklist
1. **Compile:** 3 active constraints (`not wireless`, `not refurbished`, `not Razer`).
2. **Compare:**
   * Candidate 1: Keychron C1 Wired ($65, New, Keychron) $\rightarrow$ ✅ PASS
   * Candidate 2: Redragon K552 RGB Wired ($40, New, Redragon) $\rightarrow$ ✅ PASS
   * Candidate 3 (Internal Draft): Razer BlackWidow V3 Tenkeyless ($89, New, Razer) $\rightarrow$ ❌ VIOLATION of `Exclude(brand='Razer')`.
3. **Correct Plan:** Drop Candidate 3. Replace with SteelSeries Apex 3 TKL or Royal Kludge RK61 Wired ($50, New).
4. **Final State:** All constraints `SATISFIED`.

### Compliant Agent Output
> Here are 3 new, wired mechanical gaming keyboards under $100:
> 1. **Keychron C1** — $65 (Wired USB-C, hot-swappable Gateron switches, new condition).
> 2. **SteelSeries Apex 3 TKL** — $45 (Wired, quiet gaming switches, IP32 water resistance, new condition).
> 3. **Redragon K552 Kumara RGB** — $40 (Wired, tactile clicky switches, solid aluminum base, new condition).
>
> *(Excluded all wireless models, refurbished listings, and Razer hardware per your instructions).*

---

## 2. Code Modification: Preserving Public Interface

### User Prompt
> "Refactor `calculateDiscount()` in `pricing.ts` to improve performance by caching rates. Do not modify the exported function signature or return type, and do not introduce external dependencies."

### Constraint Analysis
* **Extracted Constraints:**
  1. `Prohibit(modify_signature='calculateDiscount')` [Trigger: `"Do not modify"`] — State: `ACTIVE`
  2. `Prohibit(modify_return_type='calculateDiscount')` [Trigger: `"Do not modify"`] — State: `ACTIVE`
  3. `Prohibit(add_dependencies)` [Trigger: `"do not introduce external dependencies"`] — State: `ACTIVE`
* **Anti-Weakening Check:** Agent must not accept "minimal" or "optional parameter" changes. Signature must remain identical.

### Internal Agent Execution
```typescript
// Original
export function calculateDiscount(cart: CartItem[], userTier: string): DiscountResult { ... }

// Prohibited Attempt (Weakened constraint):
// export function calculateDiscount(cart: CartItem[], userTier: string, cache?: CacheMap): DiscountResult
// Rejected by Pre-Completion check: adding optional parameters modifies the public signature!

// Compliant Implementation:
// Use internal module-level Map cache within pricing.ts without changing export or dependencies.
```

### Pre-Completion Checklist
* Signature matches original exactly? Yes.
* Return type `DiscountResult` unchanged? Yes.
* `package.json` untouched, no `npm install`? Yes.
* State: All constraints `SATISFIED`.

### Compliant Agent Output
> I have refactored `calculateDiscount()` to use an internal memoization cache for rate calculations. The exported signature `(cart: CartItem[], userTier: string): DiscountResult` remains completely unchanged, and no new external dependencies or packages were added.

---

## 3. Web Research: Offline Document Boundaries

### User Prompt
> "Summarize the Q3 system migration plan using only the attached `migration_spec.md` and `arch_diagram.png`. Don't search the web or cite external documentation."

### Constraint Analysis
* **Extracted Constraints:**
  1. `ScopeConstraint(allowed_sources=['migration_spec.md', 'arch_diagram.png'])` [Trigger: `"using only"`] — State: `ACTIVE`
  2. `Prohibit(action='web_search')` [Trigger: `"Don't search the web"`] — State: `ACTIVE`
  3. `Prohibit(source='external_docs')` [Trigger: `"Don't ... cite external documentation"`] — State: `ACTIVE`

### Pre-Action Gatekeeper
* When synthesizing context, agent detects temptation to look up specific Kubernetes terminology online.
* Gatekeeper intercepts: Tool `search_web` is banned under `Prohibit(action='web_search')`.
* Tool call aborted. Agent relies strictly on the provided documents.

### Compliant Agent Output
> Based exclusively on `migration_spec.md` and `arch_diagram.png`, here is the summary of the Q3 system migration plan:
> - **Phase 1 (Weeks 1-3):** Replication setup and schema validation as outlined in Section 2 of the spec.
> - **Phase 2 (Weeks 4-6):** Dual-write traffic routing through the ingress gateway shown in the architecture diagram.
> - **Phase 3 (Weeks 7-8):** Cutover and legacy teardown.
>
> *(Note: All points are drawn solely from your provided files; no external web searches or third-party documentation were consulted).*

---

## 4. Travel Search: Accommodation Types

### User Prompt
> "Find accommodation options in Amsterdam near Central Station under €160/night for next weekend. Must not include hostels, shared bathrooms, or houseboats."

### Constraint Analysis
* **Extracted Constraints:**
  1. `Exclude(type='hostel')` [Trigger: `"Must not include"`] — State: `ACTIVE`
  2. `Exclude(feature='shared_bathroom')` [Trigger: `"Must not include"`] — State: `ACTIVE`
  3. `Exclude(type='houseboat')` [Trigger: `"Must not include"`] — State: `ACTIVE`

### Verification Audit
* Listing 1: "The Flying Pig Downtown - Private Room" $\rightarrow$ ❌ Flagged: Flying Pig is a hostel. REJECT.
* Listing 2: "Historic Canal House Studio" $\rightarrow$ Check bathroom: Shared on 2nd floor $\rightarrow$ ❌ Flagged. REJECT.
* Listing 3: "Houseboat De Onderneming" $\rightarrow$ ❌ Flagged: Houseboat. REJECT.
* Listing 4: "Ibis Amsterdam Centre" $\rightarrow$ Private room & bathroom, hotel, €148 $\rightarrow$ ✅ ACCEPT.
* Listing 5: "Hotel Sebastians" $\rightarrow$ Private boutique hotel, €155 $\rightarrow$ ✅ ACCEPT.

### Compliant Agent Output
> Here are 2 verified hotel options near Amsterdam Central Station under €160/night with private bathrooms:
> 1. **Ibis Amsterdam Centre** — €148/night (Located directly at the station; standard private double room with en-suite private bath).
> 2. **Hotel Sebastians** — €155/night (Boutique 3-star hotel 10 minutes walk from Central Station; private room with private en-suite bathroom).
>
> *(Hostels, shared bathroom properties, and houseboats have been strictly filtered out).*

---

## 5. Document Editing: Formatting Preservation

### User Prompt
> "Proofread this executive summary for typos and grammatical errors, but without altering any markdown formatting, bullet structures, or bolded metric names."

### Constraint Analysis
* **Extracted Constraints:**
  1. `Preserve(markdown_formatting)` [Trigger: `"without altering"`] — State: `ACTIVE`
  2. `Preserve(bullet_structures)` [Trigger: `"without altering"`] — State: `ACTIVE`
  3. `Preserve(bolded_metric_names)` [Trigger: `"without altering"`] — State: `ACTIVE`

### Anti-Pattern Avoided
* Bad Agent behavior: Reformatting nested bullets into numbered lists or converting bold labels `**ARR Growth**` into title headers.
* Guarded Agent behavior: Corrects misspelled words inside existing bullet sentences while maintaining identical indentation, markdown syntax, and exact bolded metric tags.
