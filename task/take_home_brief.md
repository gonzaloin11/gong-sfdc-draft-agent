# Take-Home Task — Gong → Salesforce Draft Agent

**Candidate:** Gonzalo Insua
**Role:** AI Builder, Revenue AI team @ Tastewise

---

## One-line summary

Build a tool-using agent that reads a mock sales call transcript and produces a structured Salesforce opportunity update draft, with explainable reasoning, typed output, and an explicit abstain path for when the transcript doesn't contain enough signal to justify an update.

## Why this matters

Tastewise AEs spend ~30 minutes per call cleaning up Salesforce afterward — MEDDPICC fields, next steps, stage changes, close-date shifts. Our Revenue AI team is building the agent that collapses this to a 30-second human review.

This task is a direct extension of the AE-pipeline-management thesis you articulated in the interview at [11:24] — *"help AEs manage their pipeline, see all the opportunities they have open, understand what are the next steps they need to take, based on the signals that we can analyze we understand are the people that are more ready to buy."* We want to see you apply the multi-agent orchestration pattern you walked us through (the wine-estate invoice workflow, the Mercado Libre + Vitagel WhatsApp agent) to this exact GTM use case.

**Honest calibration note:** you showed the strongest delivery baseline of anyone I've interviewed this week. The bar for this submission is higher than standard — Include a short Loom walkthrough, like the wine-estate screen share you did live. If it confirms, we collapse the tech-screen and move to the CRO conversation.

## What to build

An **agent** (not just an LLM call) that:

1. **Ingests** a mock Gong transcript (plain text, provided below).
2. **Extracts** structured signal:
   - Deal stage delta (e.g. *"Discovery → Solution Design"*).
   - Next step + owner + date.
   - Champion / economic buyer / detractors mentioned.
   - Risks + objections raised.
   - Suggested close-date shift (with reasoning).
3. **Drafts** a Salesforce opportunity update as a structured JSON payload matching the schema below.
4. **Explains** its reasoning — for each updated field, cite the transcript quote + timestamp that justifies the change.
5. **Knows when to abstain** — if there's no real signal (mostly small talk), it says so and proposes nothing, while still logging a last-touch summary.

### Required agentic properties

- It must use **tools** — not just one LLM prompt. At minimum: a *transcript-reader* and a *schema-validator*. Bonus: an *uncertainty-check* tool.
- It must handle **both test transcripts** (rich + thin — provided below).
- It must produce **typed output** that would pass basic schema validation against the SFDC-lite schema.

### Stack

Pick any agent framework — **Claude Agent SDK, MCP, LangGraph, LangChain, AutoGen, PydanticAI, CrewAI, hand-rolled**. Given the MCP integration you walked through for the wine estate, we'd love to see the same lens applied here: what framework + what models per step, with a rejected alternative named.

**Do not** use a real Salesforce instance or real Gong data. Mocks only.

## Deliverables

1. **Source code** (GitHub repo or zip) — minimal is fine. We want to see the agent loop, tool definitions, and the schema.
2. **A Loom walkthrough** (≤5 min) — same style as the wine-estate workflow you showed in the interview. Run the agent on both transcripts. Walk us through the reasoning traces. Show the abstain behavior on Transcript 2. *This is the high-signal artifact — prioritize it.*
3. **A write-up** (≤1.5 pages). Cover:
   - Framework choice and why — what did you reject and why?
   - Model selection per step (Opus vs. Sonnet vs. Haiku — which step gets which model, and why)?
   - How you'd evaluate this in production — eval harness concept.
   - Failure modes you'd expect in the wild + mitigations.
   - **Outcome measurement:** given your audit → MVP → measure framework, how would you measure this agent's impact on an AE's cleanup time? Rough methodology is fine.
   - What you cut and would add with another 4 hours.

## Explicit scope guardrails

- **No UI needed.** CLI or notebook is fine.
- **No deployment.** Run local.
- **No fine-tuning, no RAG, no vector DB** — wrong scope for this task.
- **Cut scope before you run over.** A clean 80% demo beats a messy 100%.

## How we'll evaluate

1. **Does the agent actually decide things?** — Tool-calling, branching, abstaining — not a chained prompt.
2. **Is the output trustworthy?** — Typed, grounded in quotes, honest about uncertainty.
3. **Is the schema handling real?** — Would your JSON pass a real SFDC validator?
4. **Production reasoning in the write-up** — Can you operate this, not just build it?
5. **Outcome measurement** — Your audit → MVP → measure methodology should show up in the write-up.

## Provided: mock SFDC-lite Opportunity schema

```json
{
  "opportunity_id": "string (required)",
  "stage": "enum: prospecting | discovery | solution_design | proposal | negotiation | closed_won | closed_lost",
  "amount_usd": "number",
  "close_date": "ISO date string",
  "next_step": {"description": "string", "owner": "string", "due_date": "ISO date string"},
  "meddpicc": {
    "metrics": "string", "economic_buyer": "string", "decision_criteria": "string",
    "decision_process": "string", "paper_process": "string", "identify_pain": "string",
    "champion": "string", "competition": "string"
  },
  "risks": ["string"],
  "last_touch_summary": "string",
  "confidence": "float 0-1"
}
```

## Provided: mock transcripts

### Transcript 1 — "Rich signal" (~40 lines, condensed)

```
[00:02] AE: Thanks for the time, Priya. Last time we left off you were going to loop in Ben from Procurement.
[00:18] Priya: Yeah — Ben's in the next call. He's fine with the $120k range but wants annual not quarterly.
[00:34] AE: Understood. And on the evaluation criteria — is speed-to-insight still #1?
[00:45] Priya: Speed is #1. Second is how well it integrates with our innovation stage-gate. We'd want that in writing.
[01:22] AE: Noted. On timing — still targeting Q3 kickoff?
[01:30] Priya: Actually pushing to early Q4 now. Our 2027 portfolio review moved.
[02:05] AE: Okay. So on my side — I'll send a revised quote tonight annual, and a one-pager on stage-gate integration by Friday. You'll loop in Ben for next Tuesday?
[02:20] Priya: Yes. Ben, me, and our CMO. CMO will be the signer.
[02:45] AE: Perfect. Any concerns from your side?
[02:55] Priya: One. I saw Spoonshot in market last week. The demo was lightweight but the price was aggressive. You'll need to be ready for that.
```

### Transcript 2 — "Thin signal" (~15 lines)

```
[00:02] AE: Hey! How was the trip?
[00:06] Champion: Oh amazing, went to Lisbon. You been?
[00:10] AE: Never. On the list.
[00:14] Champion: Highly recommend. Anyway — I know we had this on the calendar, but honestly I don't have much new. Still waiting to hear back from finance.
[00:40] AE: Totally fine. Any sense of timing?
[00:44] Champion: Maybe two weeks? I'll ping you.
[00:50] AE: Sounds good. Keep me posted.
```

The agent should produce a meaningful update for #1 and propose **no update** for #2 (while still logging the touch).

## Questions?

Email Nico at nicolas@tastewise.io. *I told you in the interview — I prefer you ask and get it right over guessing and going sideways. If a question would change the shape of what you build, ask.*

---

*Really enjoyed the conversation on Friday — the wine-estate invoice workflow screen share was the strongest live demo I've seen all week. This task is a direct extension of the thesis you articulated. Show me the same quality here.*
