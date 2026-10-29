# Reflection Brief – Harness Engineering Capstone

**Name:** Shaik Jugnu
**Date:** 2026-10-03

**Environment**
- Model(s): `claude-haiku-4-5-20251001`
- OS / Python: Linux / Python 3.13.0
- Approx. API spend: ~$0.15 USD (System 1: $0.1060 across 8 claims; System 2 & System 4 token counting and shift calls)

---

## Part 1 – Per-system

### System 1 – Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   In `/workspace/evidence/system1/traces/claim_04_neighbor_injury.jsonl`, the turn-by-turn `stop_reason` sequence is: `tool_use` -> `tool_use` -> `tool_use` -> `tool_use` -> `end_turn` across 5 turns. The decision logic is implemented in `claims_intake/loop.py` inside the loop function. On lines 103–111, `if response.stop_reason == "end_turn"` terminates execution and returns `FinalState`. On lines 113–128, `if response.stop_reason == "tool_use"` executes requested tools, appends the tool results to `working_messages`, and invokes `continue`. Any other stop reason triggers `raise UnexpectedStopReason` on line 130.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   `tests/test_antipatterns.py` checks against hardcoding a fixed loop bound or relying on string-matching heuristics instead of native API `stop_reason`. If the loop used a rigid fixed turn counter (e.g., maximum 3 turns), multi-step claims like `claim_04_neighbor_injury` and `claim_08_minor_porch_damage` (which both required 5 turns to complete investigation and route) would terminate prematurely with incomplete state, failing to issue routing decisions.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   `classify_claim` and `route_claim` in `claims_intake/tools.py` both accept claim identifiers and category parameters. Their JSON schema descriptions explicitly delineate boundaries: `classify_claim` specifies it is for preliminary category and severity determination, whereas `route_claim` is restricted to terminal queue dispatching. When a tool fails validation, returning a structured JSON error schema with typed error codes and allowable parameter values enables Claude to inspect the failed field and self-correct on the subsequent turn, whereas a plain error string often induces speculative hallucinations.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   For `claim_04_neighbor_injury` in `/workspace/evidence/system1/summary.md`, the run completed in 5 turns, consuming 17,408 input tokens and 868 output tokens for an estimated cost of $0.0217 USD over 11.5 seconds. This compares to the baseline README estimate of ~$0.020 USD and 4–5 turns. The slight variance arises from stochastic variations in tool argument elaboration and model explanation verbosity during policy lookups.

### System 2 – Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   From `/workspace/evidence/system2/budget.json`:
   - `baseline_tokens`: 38,708
   - `assembled_tokens`: 16,860
   - `reduction_pct`: 56.44%
   The `active` issue section heavily dominates the assembled context at 15,789 tokens (93.65% of the assembled budget). It is kept verbatim at the bottom boundary because unresolved dialogue requires full conversational nuance, precise timestamps, and exact user phrasing to answer immediate customer follow-ups without hallucinating details.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   The architectural rule is: **Compress resolved historical issues into concise structured summaries in the middle, while anchoring persistent entity invariants at the top and the active conversation verbatim at the bottom.** In our run:
   - Persistent invariants (`case_facts`): 204 tokens (kept byte-exact / structured)
   - Resolved Issue 1 (`resolved_refund`): compressed from 12,334 input tokens down to 414 tokens (96.6% reduction)
   - Resolved Issue 2 (`resolved_subscription`): compressed from 11,475 input tokens down to 471 tokens (95.9% reduction)
   - Active issue (`active`): 15,789 tokens (preserved 100% byte-exact)

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   In `/workspace/evidence/system2/eval.jsonl`, all test questions passed. In `/workspace/evidence/system2/eval_control.jsonl` (the ablation run where the persistent facts block was removed), **Q6 regressed to `FAIL (expected FAIL)`**, outputting: *"Based on the context provided, there is no structured status token recorded for the payment-method update issue."* This regression proves that middle-context summarization loses transient entity statuses unless they are extracted into a dedicated, top-boundary invariant facts block.

### System 3 – Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   From `.claude/rules/`:
   ```yaml
   ---
   globs: ["src/backend/**/*.py"]
   ---