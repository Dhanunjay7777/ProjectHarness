# Reflection Brief — Harness Engineering Capstone

## Environment

**Model(s):** Anthropic Claude was used by the project workflows. The final System 1 and System 4 run outputs do not record a specific model name.

**OS / Python:** Linux workspace environment, Python 3.13.0.

**Approx. API spend:** System 1 recorded an estimated total cost of **$0.1354** for run `20260925_145357`. A separate total API spend for all four systems was not recorded in the final artifacts.

---

# Part 1 — System Reflections

## 1. Loop control

The System 1 implementation makes `response.stop_reason` the control signal: the loop continues when the value is `"tool_use"` and returns when it is `"end_turn"`. Any other stop reason raises `UnexpectedStopReason`, so the loop does not rely on a hard-coded iteration count. This behavior is implemented in `system1-agentic-loop/claims_intake/loop.py`, where the response stop reason determines whether another tool-processing turn is required. My live run `20260925_145357` demonstrated this multi-turn behavior, including 2–5 turns across the eight claims.

**Artifact:** `system1-agentic-loop/claims_intake/loop.py`; run `20260925_145357`.

## 2. Anti-pattern

One explicit anti-pattern check is `test_no_integer_literal_iteration_cap_in_loop`. It protects against replacing the model's `stop_reason`-driven control flow with an arbitrary loop limit. Another related check is `test_stop_reason_is_loop_control`, which verifies that the stop reason remains the actual mechanism controlling continuation. If these protections were broken, a valid multi-turn claim could terminate because of an implementation-defined iteration cap rather than because Claude returned an appropriate stop reason.

**Artifact:** `system1-agentic-loop/tests/test_antipatterns.py`.

## 3. Tool design

Two tools can receive inputs that may look similar because they operate on claim and policy information, so their descriptions and schemas need to clearly communicate the intended use and required identifiers. The tool layer also returns structured JSON errors with `is_error: true`, including validation failures such as an invalid `policy_id` type or an unknown policy ID. This allows the model to receive an explicit tool failure and adapt instead of treating the failure as an unstructured application crash. The system prompt reinforces this by telling the model to read the error and adapt rather than retry blindly.

**Artifact:** `system1-agentic-loop/claims_intake/tools.py`; `system1-agentic-loop/claims_intake/system_prompt.py`.

## 4. Numbers

In live run `20260925_145357`, `claim_03_water_damage` used **5 turns**, with **17,352 input tokens**, **1,070 output tokens**, and an estimated cost of **$0.0227**. The complete run processed eight claims and reported a total estimated cost of **$0.1354**. The recorded results were not uniformly successful: `claim_03`, `claim_04`, `claim_05`, `claim_07`, and `claim_08` routed, while claims 01, 02, and 06 ended as incomplete. Therefore, the run provides concrete evidence of multi-turn operation and cost tracking, but it should not be described as a perfect end-to-end result.

**Artifact:** `system1-agentic-loop/runs/20260925_145357/summary.md`.

---

# System 2 — Context Strategy

## 5. Reduction

The System 2 run `20260925-141751` recorded a baseline transcript size of **38,708 tokens** and an assembled context size of **16,872 tokens**, giving a **56.41% reduction**. The largest section after assembly was the active context at **15,789 tokens**, while the case-facts block was only **204 tokens**. The active conversation remained large because it represented information still needed for the current interaction, whereas stable facts and resolved information could be represented much more compactly.

**Artifact:** `system2-context-strategy/runs/20260925-141751/budget.json`.

## 6. Summarize vs. preserve

The context strategy does not treat every section identically. Stable case facts are preserved compactly, while resolved sections are compressed because their full conversational form is no longer necessary for the active context. In the recorded run, the refund section compressed from **12,334 input tokens to 368 output tokens**, and the subscription section from **11,475 input tokens to 503 output tokens**. The resulting section sizes were **204 tokens for case facts**, **381 for resolved refund**, **516 for resolved subscription**, and **15,789 for active context**.

**Artifact:** `system2-context-strategy/runs/20260925-141751/budget.json`.

## 7. Facts block

The full evaluation run against the complete assembled context achieved **6/6 passed** (100%), while the control evaluation run against the case-facts-stripped variant produced **5/6 passed** (83.33%), demonstrating a direct and measurable regression on **Q6** (`What is the structured status of the payment-method update issue (use the exact status token from the case record, not a paraphrase)?`).

In the full assembled context, Q6 passed because the persistent `# Case Facts` block at the top boundary explicitly provided the structured token `- payment_update_status: in_progress`. In the control run where the case-facts block was stripped, Q6 regressed to **FAIL**: the model strictly adhered to the system instructions ("If the answer is not present, say 'unknown' — do not invent") and responded `"unknown"`, noting that no formal case record or structured status token was present in the remaining context. Although the active conversation discussed the ongoing effort to update the payment card, the exact snake_case token `in_progress` existed exclusively in the case-facts block. In contrast, Q1 passed even without the facts block because the compressed refund summary had retained `$22.14` verbatim. This demonstrates that while lossy narrative summaries can occasionally preserve specific figures, the structured case-facts block is load-bearing and essential for deterministic recall of machine-readable operational metadata.

**Artifact:** `system2-context-strategy/runs/20260925-141751/eval.jsonl`; `system2-context-strategy/runs/20260925-141751/eval_control.jsonl`.

---

# System 3 — Claude Code Configuration

## 8. Path-scoped rules

The API-specific rule uses the frontmatter glob `paths: ["src/api/**/*"]`. This makes the convention apply specifically to API code rather than forcing every part of the repository to inherit an API-specific rule. For a cross-cutting convention, path-scoped rules are preferable to adding increasingly broad directory-level `CLAUDE.md` files because the rule can target the files where the convention actually matters.

**Artifact:** `system3-claude-code-config/.claude/rules/...` API rule with `paths: ["src/api/**/*"]`.

## 9. Forked skill

The deployment-check skill declares `context: fork` and restricts its allowed tools to read-oriented operations such as `Read`, `Grep`, `Glob`, and selected read-only Git/GitHub commands. Forking gives the skill an isolated context for performing the deployment check, while the restricted tool set prevents the check from silently changing or deploying repository content. The skill explicitly avoids write, push, and deployment operations. Without these boundaries, a diagnostic deployment check could have a much larger operational blast radius.

**Artifact:** `system3-claude-code-config/.claude/skills/...` deployment-check skill.

## 10. Scope

The project-level scope is represented by `./CLAUDE.md`, together with project `.claude` standards and rules. User-level configuration is represented by the `~/.claude/` layer, while directory-level behavior can be associated with a specific directory or its `CLAUDE.md`. The configuration therefore distinguishes repository-specific instructions from personal/global Claude Code behavior.

**Artifact:** `system3-claude-code-config/CLAUDE.md`; Claude Code configuration scope table.

---

# System 4 — Orchestration

## 11. Push work down

The warm tier stores defects in SQLite and exposes queries such as `defects_since`, rather than sending the complete historical defect database to the model. The implementation uses an indexed query of the form `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, allowing the orchestration layer to retrieve only the relevant recent records. This means the model receives the selected evidence instead of having to reason over the entire historical store. In my live Shift A run, the resulting new-defect count was **0**, so I am not claiming a runtime historical query count that was not actually recorded.

**Artifact:** `system4-orchestration/shift_monitor/warm.py`; live Shift A result: `0 new defects`.

## 12. Crash recovery

The recovery implementation uses a **30-minute stale-resume threshold**. An incomplete recent state can be resumed when it is within the threshold, while an older incomplete state is treated as stale and starts fresh. Starting fresh with a compact summary can be more reliable than continuing from an old state whose assumptions may no longer match the current operational situation.

**Artifact:** `system4-orchestration/shift_monitor/recovery.py`, where `STALE_RESUME_THRESHOLD_MINUTES = 30`.

## 13. Small state

The hot-state budget is **5,120 bytes**, and the live Shift A run produced a `data/hot_state.json` of only **130 bytes**. The generated state contained recent defect hashes, a current-shift summary, active alerts, and threshold statuses. Keeping this state bounded prevents the persistent cross-session context from growing indefinitely and protects future model calls from accumulating unnecessary historical information.

**Artifact:** `system4-orchestration/data/hot_state.json`; `system4-orchestration/shift_monitor/state.py`.

---

# Part 2 — Synthesis

## 14. Three layers

The four systems demonstrate three complementary layers of harness engineering. **Files/artifacts** provide durable information such as claim fixtures, context state, Claude Code rules, and warm-tier defect data. The **harness** provides deterministic control over model interaction, tool execution, validation, context assembly, and state budgets. **Orchestration** connects those pieces across turns or sessions so that the model does not have to manage all persistence and control logic itself.

**Artifact:** Systems 1–4 implementation directories and their respective test suites.

## 15. Deterministic vs. prompt behavior

Code should guarantee behaviors where correctness must not depend on model interpretation: loop termination, tool validation, state-size limits, atomic writes, recovery thresholds, and allowed tools are examples. Prompts are better suited to behaviors requiring language understanding, such as deciding what information is relevant or explaining a result. The architecture works because deterministic enforcement handles the safety and reliability boundaries while the model handles the reasoning tasks inside those boundaries.

**Artifact:** `system1-agentic-loop/claims_intake/loop.py`; `system4-orchestration/shift_monitor/state.py`; System 3 deploy skill.

## 16. Context's two faces

System 2 manages context **within a long conversation**, reducing a **38,708-token** baseline to an assembled **16,872-token** context, a **56.41% reduction**. System 4 manages context **across operational sessions**, keeping a compact hot state whose live file size was only **130 bytes** while historical defects remain in a warm SQLite store. The first approach controls the size and usefulness of an active conversation; the second controls what needs to survive between shifts.

**Artifact:** System 2 run `20260925-141751`; System 4 `data/hot_state.json`.

## 17. Reliability invisible in one run

A single successful model run cannot prove that an architecture is reliable. The test suites provide stronger guarantees for behaviors such as stop-reason loop control, anti-pattern prevention, context assembly, Claude Code configuration, crash recovery, atomic state handling, and orchestration behavior. In the final validation, System 4 alone produced **33 passing tests**, while System 3 produced **35 passing tests**, demonstrating that important reliability properties are verified independently of one particular model response.

**Artifact:** System 3 validation: `35 passed`, `OK`; System 4 validation: `33 passed`.

## 18. Blast radius

System 3 has a particularly explicit blast-radius boundary: the forked deployment-check skill is limited to read-oriented tools and does not permit push or deployment actions. System 4 similarly limits state growth through the 5,120-byte hot-state budget and uses atomic replacement when writing state. System 1 limits tool failures to structured errors and makes loop behavior dependent on explicit stop reasons. These mechanisms act as practical kill switches or containment boundaries because an individual model decision cannot automatically expand into unrestricted repository changes, unbounded state, or uncontrolled execution.

**Artifact:** System 3 deploy-check skill; System 4 `state.py`; System 1 `tools.py` and `loop.py`.

---

# Part 3 — Honest Reflection

## 19. What broke

The first System 1 live execution failed before the harness could run because `anthropic==0.39.0` was incompatible with the installed `httpx==0.28.1`; the Anthropic client attempted to pass the removed `proxies` argument. I fixed the environment by pinning `httpx` to **0.27.2**, after which the same System 1 live workflow executed successfully and produced run `20260925_145357`. The successful run still exposed incomplete claim outcomes, which is useful evidence that environment compatibility and application-level behavior are separate reliability concerns.

**Artifact:** System 1 environment fix; run `20260925_145357`.

## 20. What I would change

If I rebuilt the capstone, I would make dependency compatibility part of the reproducibility setup rather than discovering it during the final live run. In particular, the System 1 environment should explicitly constrain the compatible `httpx` version alongside its Anthropic dependency, because the original environment allowed a version combination that prevented the client from initializing. I would also preserve a deterministic recorded-response mode for final demonstrations so that architectural behavior can be reproduced without depending on external API availability, while still keeping separate live-run evidence when API execution is required.

**Artifact:** `system1-agentic-loop/pyproject.toml`; installed `httpx==0.27.2`; live run `20260925_145357`.

---

## Final Evidence Summary

* **System 1:** live run `20260925_145357`; 8 claims; **$0.1354 estimated total cost**.
* **System 2:** **30 passed**; full evaluation **6/6 passed**; control evaluation **5/6 passed** (Q6 regressed); **56.41% context reduction**.
* **System 3:** **35 passed**; configuration validator **OK**.
* **System 4:** **33 passed**; live Shift A completed with **0 new defects**; hot state **130 bytes** against a **5,120-byte budget**.
* The System 1 live run contained incomplete outcomes for claims 01, 02, and 06; these results are intentionally retained in the reflection rather than omitted.
* The System 2 control run artifact (`eval_control.jsonl`) contains all 6 question evaluations, demonstrating 5/6 passed with an expected regression on Q6 (structured status token `in_progress` missing when the case-facts block is removed), confirming that the case-facts block is load-bearing.
