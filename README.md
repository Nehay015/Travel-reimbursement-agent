# Travel Reimbursement Approval Agent — Workflow & README

**Candidate submission** · GenAI / Agentic AI Developer role · HCL Technologies
**Deliverable:** `yourname_TravelReimbursementAgent.ipynb` (single, self-contained, runnable notebook)

This file documents *what was built, why, and how to run it* — a companion to the in-notebook
README/Design Notes, meant to sit at the root of the GitHub repo so a reviewer understands the
project before opening the notebook.

---

## 1. What this is

A working prototype of an AI-assisted agent that takes an employee travel reimbursement claim,
grounds itself in a company travel policy, runs the claim through a set of purpose-built tools,
and returns a structured decision: `APPROVE`, `PARTIAL_APPROVE`, `REJECT`, or `MANUAL_REVIEW`.

Built against the exact scope given in the assignment PDF — the policy in **Appendix A** and the
5 sample claims in **Appendix B** — nothing invented, no extra claims added.

---

## 2. Workflow — what was actually done, step by step

1. **Read the assignment PDF** and extracted the two graded inputs verbatim:
   - Appendix A (policy, with stable `POL-*` rule IDs)
   - Appendix B (5 sample claims)
2. **Modeled the policy as structured data** (a Python dict keyed by rule ID) instead of free text,
   so the agent can retrieve and cite specific rules — a lightweight stand-in for a RAG/context
   store, sized appropriately for ~15 rules (a vector DB would be over-engineering here).
3. **Built 6 single-purpose tools**, each independently testable, each returning structured
   (not free-text) output:
   - `policy_lookup` — retrieve a rule by ID (grounding)
   - `receipt_completeness_check` — POL-RCT-01/02
   - `per_diem_limit_checker` — POL-CAT-02, POL-PD-01/02/03, POL-AIR-01
   - `duplicate_detector` — simple in-run signature match
   - `approval_threshold_check` — POL-APR-01/02/03
   - `output_validator` — schema check with fail-safe downgrade to `MANUAL_REVIEW`
   - (plus `timeliness_check` for POL-TIME-01, used inside the pipeline)
4. **Built the Agent Orchestrator** — a fixed-order ReAct-style pipeline that calls the tools,
   combines their outputs, applies a deterministic decision policy (any exception/ambiguity found
   by any tool routes the claim to Manual Review — never forced), and prints a full tool-call
   audit trail per claim.
5. **Added a GenAI explanation layer** — after the decision is made deterministically, an LLM (or a
   local deterministic fallback template, if no API key is configured) generates the natural-language
   `explanation` and a heuristic `confidence` score, grounded only in the tool outputs. The LLM never
   computes money — see Section 4 below for why.
6. **Ran the agent over all 5 sample claims**, printed the audit trail and 3+ sample outputs, and
   emitted the required structured JSON array in the final code cell.
7. **Built the dashboard** — a decision-breakdown chart and an approved-vs-deducted-per-claim chart,
   rendered inline under a `## Dashboard` heading and also saved as `UI_SS_1.png`.
8. **Verified correctness by hand** against Appendix A for all 5 claims before treating the notebook
   as done (see Section 5).
9. **Executed the notebook top-to-bottom** with `nbclient` to confirm zero errors and reproducible
   output before submission.
10. **(Optional extension, separate file)** Wrote `azure_gpt4o_integration.py` — a standalone
    reachability/diagnostic script plus a drop-in `azure_chat_complete()` function, so the
    explanation layer can optionally be backed by an Azure-hosted GPT-4o deployment instead of
    OpenAI/Anthropic/local-simulated, without hardcoding credentials into the notebook.

---

## 3. How to run

```bash
pip install pandas matplotlib          # required
pip install openai anthropic requests  # optional — only needed if you want a real LLM call
jupyter notebook yourname_TravelReimbursementAgent.ipynb
```

Then **Run All**. No API key is required — the notebook auto-detects `OPENAI_API_KEY` /
`ANTHROPIC_API_KEY` and falls back to a local, deterministic explanation template if neither is
set, so it runs identically for any reviewer on any machine.

**Optional — Azure GPT-4o instead of OpenAI/Anthropic:**
```bash
export AZURE_OPENAI_ENDPOINT="https://<your-resource>.openai.azure.com"
export AZURE_OPENAI_DEPLOYMENT="<your-deployment-name>"
export AZURE_OPENAI_API_VERSION="2024-02-01"
export AZURE_OPENAI_API_KEY="<your-key>"
python azure_gpt4o_integration.py     # run this first to confirm connectivity
```
If it reports `ok: true`, `azure_chat_complete()` can be dropped into the notebook's
`call_llm_for_explanation()` as a third provider branch, following the same
try/except-with-fallback pattern already used for OpenAI and Anthropic.

---

## 4. Key design decision: deterministic tools + LLM-as-explainer

Money-affecting logic (per-diem caps, receipt rules, approval tiers, and the final decision
itself) is implemented as **plain, deterministic Python** — not left to an LLM to compute — because:

- **No hallucinated numbers.** A wrong dollar figure in a reimbursement pipeline is a compliance
  problem, not just an unhelpful answer.
- **Auditability.** Every deduction or Manual Review routing traces to a specific `POL-*` rule ID,
  reproducibly, every run.
- **Reproducibility for grading.** The same 5 claims produce identical decisions on any machine,
  with or without an API key.

The LLM's role is scoped to what GenAI is actually good at here: turning structured tool output
into a clear, grounded explanation, and estimating a confidence score. This trade-off (fixed-order
tool pipeline vs. true LLM function-calling) is discussed explicitly in the notebook's
"Design Notes & Reasoning" section, along with what a next iteration would look like.

---

## 5. Verified results (5/5 claims, hand-checked against Appendix A)

| Claim | Decision | Approved | Deducted | Why |
|---|---|---|---|---|
| CLM-001 | `APPROVE` | $1,110.00 | $0.00 | All items eligible, in-policy, receipts present |
| CLM-002 | `REJECT` | $0.00 | $380.00 | Spa + minibar — both ineligible (POL-CAT-02) |
| CLM-003 | `PARTIAL_APPROVE` | $840.00 | $100.00 | Lodging over nightly cap (POL-PD-02) |
| CLM-004 | `MANUAL_REVIEW` | — | — | Business-class airfare exception (POL-AIR-01) + missing lodging receipt (POL-RCT-02) + total > $2k (POL-APR-03) |
| CLM-005 | `MANUAL_REVIEW` | — | — | Missing required meal receipt (POL-RCT-02) |

---

## 6. Repo contents

| File | Purpose |
|---|---|
| `yourname_TravelReimbursementAgent.ipynb` | **Main deliverable** — the full agent, run top-to-bottom |
| `UI_SS_1.png` | Dashboard screenshot (also rendered inline in the notebook) |
| `README.md` | This file |
| `azure_gpt4o_integration.py` | Optional: Azure GPT-4o connectivity check + drop-in LLM call (not required by the assignment; included as an extension) |

---

## 7. Assumptions & limitations

- All 5 claims are embedded exactly as given in Appendix B; the same `run_agent_on_claim()`
  function works unchanged on any new claim following the same JSON shape.
- Meals/ground-transport per-diem caps are applied to the claim's aggregate line amount using the
  stated day count (the sample claims report these as one aggregated line, not per-day lines).
- `duplicate_detector` uses a simple in-memory signature for this run only; a production version
  would check a persistent claims database with fuzzier matching.
- Confidence scores are heuristic, not a calibrated model output.
- This is a prototype: no auth, no persistent storage, no concurrency handling — intentionally,
  given the 2–3 day timebox and the assignment's explicit "don't over-engineer" guidance.

## 8. What I'd improve next

- Real function-calling (OpenAI/Anthropic/Azure tool-use API) so the LLM actively chooses tool
  order at runtime instead of following a fixed pipeline.
- A persistent audit-trail store (append each claim's tool trace to a log/DB) for compliance review.
- Automated test cases (pytest) asserting each tool's output against hand-computed edge cases.
- A feedback loop that captures Manual Review outcomes to refine routing rules over time.
