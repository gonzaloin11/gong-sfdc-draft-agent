# Gong → Salesforce Draft Agent

**Take-home submission — AI Builder, Tastewise Revenue AI team**
**Author:** Gonzalo Insua

A tool-using agent that reads a mock Gong sales-call transcript and produces a structured Salesforce opportunity update — with citations to the transcript, schema-validated output, and an explicit abstain path when the call has nothing actionable.

Built as four n8n workflows: one main orchestrator (two LLM agents) and three deterministic tools (Code-node sub-workflows).

**▶ Loom walkthrough:** https://www.loom.com/share/4a82cb6b825b4b5e9ab742010dbb6da6

---

## Architecture (v6 — final)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Phase 1 — Foundation                                │
│  Manual Trigger → raw_transcripts_source → text_pre_processor               │
│                                          → transcript_reader (per-item)     │
│                                                                             │
│                         Phase 2 — Per-OPP loop                              │
│  loop_over_items (splitInBatches:1)                                         │
│       ↓ process branch                                                      │
│  extraction_agent (Haiku 4.5, NO tools)         ← drafts JSON only          │
│       ↓                                                                     │
│  extraction_post_normalizer                     ← parses, builds            │
│                                                   evidence_list             │
│                                                                             │
│                         Phase 3 — Tool-using reviewer                       │
│  review_agent (Sonnet 4.5, 3 ai_tools)          ← THIS IS THE AGENT         │
│       ├── quote_grounding  ← deterministic substring checker                │
│       ├── schema_validator ← deterministic SFDC-lite validator              │
│       └── uncertainty_check← deterministic heuristic abstain reviewer       │
│       ↓                                                                     │
│  review_post_normalizer (5-point quality_score)                             │
│                                                                             │
│                         Phase 4 — Branching gate                            │
│  IF quality_score >= 4  → finalize_from_review (use agent's verdict)        │
│  ELSE                   → call_quote_grounding → ... → merge_validations    │
│                           (deterministic fallback chain — same 3 tools,     │
│                            invoked deterministically as a safety net)       │
│       ↓                                                                     │
│  unify → rate_limit_wait (65s) → loop back                                  │
│                                                                             │
│  loop done → final_aggregator (run_summary, by_path counts, all results)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Why two agents

The reviewer agent **does not see the transcript**. Its inputs are: the extraction agent's draft payload, the reasoning_trace (claims with quotes), and the names of three tools. To verify the draft, it **must** call tools — the tools are the only source of ground truth. This makes tool-use *structurally necessary*, not just instructed. The agent's branching decision (approve / revise / abstain) is read from tool results.

For abstain cases, the agent skips `quote_grounding` entirely (nothing to ground in an abstain) and only calls `schema_validator` + `uncertainty_check`. That's branched tool-calling driven by input state — not a fixed script.

### Why the deterministic fallback exists

If the review_agent returns malformed output or skips its tools, the `review_post_normalizer` scores the verdict on 5 deterministic checks (parseable, valid action, payload shape, tools_invoked, internally_consistent). Score < 4 → IF gate routes to the deterministic chain that re-runs the same tool logic without the agent. Per OPP, **exactly one path executes** (not both). The system always ships correct output; the path taken is logged in `meta.path_taken`.

In our test runs, both transcripts went via `review_agent` with quality_score 5/5 — the deterministic fallback was never needed but stayed as the floor.

---

## Repository layout

```
n8n builder/
├── README.md                          ← this file
├── WRITEUP.md                         ← framework choice, models per step,
│                                       eval harness, failure modes,
│                                       outcome measurement (1.5 pages)
├── Claude.md                          ← project instructions for Claude Code
│
├── task/
│   └── take_home_brief.md             ← verbatim brief from Tastewise
│
├── transcripts/
│   ├── transcript_1_rich.txt          ← 10-utterance rich transcript
│   └── transcript_2_thin.txt          ← 7-utterance thin (abstain) transcript
│
├── workflows/                         ← n8n workflow JSONs (import these)
│   ├── main_gong_sfdc_agent.json      ← main orchestrator (30 nodes)
│   ├── tool_quote_grounding.json      ← anti-hallucination deterministic tool
│   ├── tool_schema_validator.json     ← SFDC-lite schema validator
│   └── tool_uncertainty_check.json    ← heuristic abstain reviewer
│
├── schemas/
│   └── opportunity_update.json        ← JSON Schema (draft-07) for the output
│
├── outputs/
│   └── sample_run.json                ← actual end-to-end run output, both OPPs
│
└── docs/
    ├── ERRORS_AND_FIXES.md            ← every bug we hit + how we fixed it
    └── SOPs/                          ← reusable subagent prompts
        └── export-workflow-json.md
```

---

## Sample output (real run, no edits)

**Transcript 1 (rich) — agent approves a full payload after grounding 9 quotes:**

```json
{
  "opportunity_id": "OPP-001",
  "abstain": false,
  "payload": {
    "stage": "proposal",
    "amount_usd": 120000,
    "close_date": "2026-10-15",
    "next_step": {
      "description": "Send revised annual quote and one-pager on stage-gate integration; loop in Ben and CMO",
      "owner": "AE",
      "due_date": "2026-05-02"
    },
    "meddpicc": {
      "economic_buyer": "CMO",
      "decision_criteria": "Speed-to-insight; stage-gate integration",
      "decision_process": "Ben (Procurement), Priya, CMO sign-off",
      "paper_process": "Annual contract; integration requirements in writing",
      "champion": "Priya",
      "competition": "Spoonshot",
      "metrics": "",
      "identify_pain": ""
    },
    "risks": [
      "Spoonshot aggressive pricing",
      "Stage-gate integration must be documented to close"
    ],
    "last_touch_summary": "Priya confirmed $120k annual, early Q4 close, CMO as economic buyer and signer; Spoonshot competitive threat identified.",
    "confidence": 0.82
  },
  "validation": {
    "tool_summary": {
      "quote_grounding": { "all_grounded": true, "failed_fields": [] },
      "schema": { "valid": true, "errors": [] },
      "uncertainty": { "signal_strength": "strong", "overrides_sonnet": false }
    }
  },
  "meta": { "path_taken": "review_agent", "review_quality_score": 5 }
}
```

**Transcript 2 (thin) — agent abstains; payload is null except for last_touch_summary:**

```json
{
  "opportunity_id": "OPP-002",
  "abstain": true,
  "abstain_reason": "Abstain confirmed: schema valid for abstain mode, uncertainty_check confirms should_abstain=true with weak signal (smalltalk_detected, vague_commitment).",
  "payload": {
    "stage": null, "amount_usd": null, "close_date": null,
    "next_step": null, "meddpicc": null, "risks": null,
    "last_touch_summary": "Champion waiting on finance feedback, vague two-week follow-up.",
    "confidence": 0.15
  },
  "reasoning_trace": [],
  "meta": { "path_taken": "review_agent", "review_quality_score": 5 }
}
```

Full run output (both OPPs) is in [outputs/sample_run.json](outputs/sample_run.json).

---

## How to import

These workflows were built and tested on n8n **v1.121** (self-hosted, Elestio). They should import on any n8n ≥1.95 that ships the langchain cluster nodes.

### Prerequisites

1. n8n instance with the `@n8n/n8n-nodes-langchain` package installed (default in modern n8n).
2. An **Anthropic API credential** added to n8n. Models used:
   - `claude-haiku-4-5` for extraction (fast, cheap, JSON drafting only)
   - `claude-sonnet-4-5` for review (better tool-call adherence; needs ~30K ITPM headroom)

### Import order (matters)

Sub-workflows must exist before the main workflow can reference them:

```
1. Settings → Workflows → Import from File →  workflows/tool_quote_grounding.json
2. Settings → Workflows → Import from File →  workflows/tool_schema_validator.json
3. Settings → Workflows → Import from File →  workflows/tool_uncertainty_check.json
4. Activate all three sub-workflows (toggle "Active" on each).
5. Settings → Workflows → Import from File →  workflows/main_gong_sfdc_agent.json
```

### Post-import wiring (one-time)

1. Open the **main workflow**. The two LLM nodes (`extraction_model`, `review_model`) will show "Credential not set" — pick your Anthropic credential from the dropdown. The credential field was stripped on export so the JSON isn't tied to one instance.

2. The three `toolWorkflow` nodes inside the main workflow (`tool_quote_grounding`, `tool_schema_validator`, `tool_uncertainty_check`) reference sub-workflows by ID. After import, the IDs will be the new ones n8n assigned. Open each and pick the matching sub-workflow from the dropdown.

3. The two `executeWorkflow` nodes in the deterministic-fallback chain (`call_quote_grounding`, `call_schema_validator`, `call_uncertainty_check`) — same fix-up: re-pick the sub-workflow.

### Run

Click **Execute Workflow** on the main workflow. The two transcripts are pinned in the `raw_transcripts_source` Set node so no input is required.

Expected runtime: ~2–3 minutes (one LLM round per OPP × 2 OPPs + 65s rate-limit wait between iterations to stay under Sonnet's 30K ITPM ceiling).

### Optional: rate-limit tuning

If your Anthropic tier has a higher Sonnet ITPM budget, you can drop `rate_limit_wait.amount` from 65 to 5–30 seconds for faster demos.

---

## Loom walkthrough

**▶ Watch:** https://www.loom.com/share/4a82cb6b825b4b5e9ab742010dbb6da6

The Loom shows:
1. n8n editor view of the architecture (the topology above, real nodes).
2. One Execute click → both OPPs flowing through.
3. Inside `review_agent`, the three tool-call sub-executions visible per OPP-001 and the **two** tool-call sub-executions for OPP-002 (skipped grounding because abstain). This is the agent branching its tool-calling pattern.
4. Final aggregator output: 1 update + 1 abstain, both via `review_agent` path, quality_score 5/5 each.
5. Quick walk through `validation.tool_summary` to show the agent reads tool results before deciding.

---

## What I'd add with another 4 hours

See [WRITEUP.md](WRITEUP.md). Top three:
- A real eval harness (golden-set JSON × N transcripts, computing field-F1 / abstain-accuracy / grounding-pass-rate / agent-path adherence).
- Stricter `quote_grounding` that does value-presence checks, not just substring matches.
- Webhook trigger + Python CLI runner for programmatic invocation.


---

## What's documented elsewhere

- **Bug log + lessons:** [docs/ERRORS_AND_FIXES.md](docs/ERRORS_AND_FIXES.md). 11 sections covering every concrete failure we hit (Anthropic ITPM, Haiku draft-only behavior, structuredOutputParser interception, the chained-prompt trap, n8n MCP limitations, AI-fillable schema gotcha, `$fromAI` syntax, Code node `runOnceForAllItems` mode silently dropping items, `$('Node').first()` returning the wrong loop iteration, empty-array tool inputs being rejected, citation quality vs. proximity).

- **Reusable subagent prompts:** (IN PROGRESS) [docs/SOPs/](docs/SOPs/). Each SOP describes when to use it, the prompt template, edge cases, and the date last verified.
