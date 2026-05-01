# Errors Encountered & Fixes — Build Log

A working journal of every error we hit while building the Tastewise Gong→SFDC agent, what caused it, and how we fixed it. Written for future-me / future-collaborators so the same problems don't bite twice.

---

## 1. Anthropic ITPM Rate-Limit Errors (the big one)

### Symptom

```
The service is receiving too many requests from you
This request would exceed your organization's rate limit of 30,000 input tokens per minute
(model: claude-sonnet-4-5-20250929 / claude-haiku-4-5-20251001)
```

Workflow execution canceled mid-run, sometimes after 75s, sometimes immediately on the second OPP.

### Why it happens (the part that took us hours to understand)

ITPM means **input tokens per minute**, measured on a **rolling 60-second window**, summed across **every API call** to that model. Two things people get wrong:

1. **It is not "tokens per call."** It's the sum of input tokens across all calls in the last 60 seconds. Two calls of 25K each at t=0 and t=20s = 50K in the bucket. You hit the limit even though no single call was big.
2. **Per-tier limits differ wildly between models.** Haiku 4.5 was 50K ITPM on this account; Sonnet 4.5 was **30K ITPM**. We blew past Sonnet's ceiling without realizing it had a tighter ceiling than Haiku.

The other compounder is the **n8n LangChain agent loop**: every tool call inside the agent re-sends the full prior context (system prompt + user prompt + all prior assistant turns + all prior tool results) on the next API call. A 5-round-trip agent loop with a 7K-token initial context burns ~30K input tokens **per OPP** even before tools.

### Things we tried that didn't work alone

| Attempt | Why it failed |
|---|---|
| Compress system prompt 3500→1000 tokens | Helped a bit; not enough on its own. |
| Increase agent retries (`maxTries: 5, waitBetweenTries: 1500ms`) | **Made it worse** — every retry re-sends the full input → multiplies ITPM consumption. |
| Set `rate_limit_wait` to 5s between OPPs | Way too short. Rolling window is 60s; 5s means OPP-2 runs while half of OPP-1's tokens still count. |
| Set `rate_limit_wait` to 30s | Better but inconsistent. Sometimes worked for Haiku, but Sonnet's lower ceiling needed more headroom. |
| `max_tokens` cap (output limit) | Helps but only marginally — Anthropic reserves *some* of `max_tokens` against your input budget, but the dominant cost is genuine input tokens. |

### What actually worked (cumulative)

1. **`maxTries: 1, retryOnFail: false`** on the agent node. Kills the retry storm. **Single biggest fix.** A failed call is better than 5 retries that exhaust ITPM and then still fail.
2. **`rate_limit_wait` ≥ 65s** between OPPs when using Sonnet. The rolling window is 60s; 65s gives a 5s jitter buffer for the bucket to clear.
3. **Slim tool return payloads.** When `quote_grounding` succeeds, return only `{all_grounded:true, summary}` — drop the verbose `results[]` array with its `matched_text` echoes. That `matched_text` echoes the source utterance back into the agent's context on every subsequent round trip.
4. **Pre-bind heavy inputs in tool nodes.** This is the architectural fix. Instead of letting the agent fill in `utterances` (a 1.5K-token array) on every tool call, pre-bind it via an n8n expression on the tool node:
   ```
   workflowInputs.value.utterances = "={{ $('extraction_post_normalizer').first().json.utterances }}"
   ```
   The agent only fills in small fields (`evidence_list`, `draft_payload`, `abstain_mode`). The big inputs come from upstream automatically. Saves ~1.5K tokens *per tool call* and the agent never has to write them out in its tool_use blocks. Across 3 tools × ~5 round trips = ~22K tokens saved per OPP.

### The n8n docs reference that helped

> **Enable "Retry on Fail"**: Open the node's settings and toggle on Retry On Fail. Set the "Wait Between Tries (ms)" to a value longer than the service's rate limit.
>
> **Use "Loop Over Items" and "Wait" Nodes**: Instead of sending all items at once, use a Loop Over Items node to batch data and a Wait node to introduce pauses between each API call.

This was directionally right but incomplete — it doesn't mention that **retries on rate-limited calls re-spend the input budget**. If you're going to retry on 429, the wait between tries has to be ≥ the rolling window of the rate limit. For Anthropic, that's 60s. Setting `waitBetweenTries: 1500ms` (the default) is actively harmful for ITPM errors.

### Diagnostic checklist for next time

When you see a 429:

1. **Check which limit you hit.** The error message names the model and the limit (e.g., "30,000 input tokens per minute"). Note the exact number.
2. **Compare against your tier.** Different models have different ceilings on the same tier.
3. **Count your usage:** rough budget per OPP = (system prompt tokens) + (user prompt tokens) × (number of agent round trips) + Σ(tool result tokens). If that × N OPPs in 60s > your limit, you'll 429.
4. **Disable retries first.** Always. Then add the wait. Then slim tool returns. Then pre-bind heavy inputs.
5. **Don't reach for "upgrade the tier" as the first answer.** It costs money and obscures the architectural issue.

---

## 2. Agent emits JSON inline instead of calling tools (Haiku draft-only behavior)

### Symptom

`extraction_agent` (Haiku 4.5) wrote a 3K-character JSON payload as plain text in its response, ending with the literal word "Now" — meaning it was about to call a tool but stopped. None of the `tool_*` nodes had `itemsInput > 0`. Quality was correct but the architecture was undermined: the agent wasn't actually using its tools.

### Why it happened

Two compounding causes:

1. **`max_tokens: 1500` was too low.** The agent ran out of output budget mid-response, was cut off, never got to emit the tool_use block. Expanding to `max_tokens: 4000` partially helped — but exposed the second cause.
2. **Haiku 4.5 has weak tool-call adherence in n8n's LangChain wrapper.** Even with explicit "you MUST call quote_grounding before emitting JSON" instructions, Haiku tended to draft the JSON inline and skip the tool. Sonnet 4.5 is materially better at tool-calling adherence under the same prompt.

The n8n quirk that compounds this: **`@n8n/n8n-nodes-langchain.lmChatAnthropic` does NOT expose Anthropic's `tool_choice` parameter** (`{"type": "any"}` to force any tool, `{"type": "tool", "name": "..."}` to force a specific tool). So we can only nudge with prompt language, not enforce.

### What we tried

| Attempt | Result |
|---|---|
| Strengthen prompt: "DO NOT emit JSON before calling tools" | Haiku still drafted inline. |
| Raise `max_tokens` to 4000 | Agent finished its draft cleanly, then sometimes called tools, sometimes didn't. Inconsistent. |
| Switch to Sonnet 4.5 | Sonnet calls tools reliably. **The fix.** |

### Architectural workaround (independent of the model fix)

When the LLM-driven agent loop is unreliable, the fallback is a **deterministic post-pipeline**: agent emits a draft, then deterministic Code+ExecuteWorkflow nodes run validation/grounding/abstain checks afterward. This is what we called "Path A." It works perfectly **but fails the take-home's evaluation criterion** ("not a chained prompt"), so we abandoned it as a primary architecture and kept it only as a conditional fallback (see #4 below).

### Lesson

If the architecture requires reliable tool-calling and you're forced to use Haiku for cost reasons, you need either (a) to upgrade to Sonnet for that specific step, or (b) to design the agent's job so tool-use is *structurally necessary* (no transcript visibility → must call tools to read truth), or (c) to accept the LLM as a single-turn JSON emitter and run validation deterministically downstream.

---

## 3. n8n agent looped 10 times without ever calling a tool (Output Parser interception)

### Symptom

`extraction_agent.error: "Max iterations (10) reached. The agent could not complete the task within the allowed number of iterations."` Tool nodes had zero invocations.

### Why it happened

The `@n8n/n8n-nodes-langchain.outputParserStructured` node was wired to the agent. When the agent emitted its first response, the parser tried to validate it against the strict JSON schema, the response failed validation (because the agent was about to call a tool, not emit final JSON), and n8n's agent forced a retry. The agent kept retrying parse-failures, never reaching the tool-calling phase, until it hit `maxIterations: 10`.

This is the **classic LangChain structuredOutputParser + tool-calling conflict**: the parser wants shape-perfect JSON on the first reply; the tool-calling pattern wants the agent's first reply to be a tool_use block. The two instructions conflict; the parser's retry-loop wins, the agent never gets to use tools.

### The fix

Remove the `Structured Output Parser` node entirely and set `parameters.hasOutputParser: false` on the agent. The downstream `extraction_post_normalizer` (a Code node) handles parsing with multiple fallback strategies: try strict JSON.parse, try after stripping markdown fences, try extracting the outermost `{...}` block, try regex-finding a fenced block. That gives us robust parsing without blocking tool-calling.

### Lesson

**Don't combine `structuredOutputParser` with `ai_tool` connections on the same agent.** Pick one:
- Schema enforcement → no tools, single-shot JSON emit, parser validates output.
- Tool-calling agent → no parser, downstream Code node parses the agent's free-form output.

The two patterns don't compose in n8n's LangChain wrapper.

---

## 4. The "chained prompt" architecture trap

### Symptom

We built a clean, reliable workflow (Path A) that produced correct output: `agent emits draft → quote_grounding (deterministic) → schema_validator (deterministic) → uncertainty_check (deterministic) → IF abstain → final`.

But it failed the take-home's #1 evaluation criterion: *"Does the agent actually decide things? — Tool-calling, branching, abstaining — not a chained prompt."*

### Why it failed that criterion

The LLM made exactly one decision (the JSON draft). Everything after was deterministic plumbing. The agent never read a tool result. The agent never branched on uncertainty. The agent never chose to abstain — the deterministic IF did. From the reviewer's seat: this is a chained prompt with extra steps, not a tool-using agent.

### The fix (current architecture)

Add a **second LLM (`review_agent`)** whose job is *only* to verify the first agent's draft using tools, and which **structurally cannot do its job without calling tools** (its prompt does not include the transcript — only the draft, the reasoning_trace, and tool descriptions). The review_agent reads tool results and emits one of `{action: "approve" | "revise" | "abstain"}` based on what the tools returned.

Behind the review_agent, we kept the deterministic Path A chain as a **conditional fallback**: a 5-point quality_score on the review_agent's verdict gates an IF — if score ≥ 4, use the agent's verdict; if score < 4, fall back to deterministic. Per OPP, exactly one branch fires. Demo is honest.

### Lesson — what makes an agent "actually decide things"

The reviewer's distinction between "tool-using agent" and "chained prompt" comes down to: **does the LLM read a tool result and choose its next action based on that result?**

- Drafting JSON → running deterministic validators afterward = chained prompt.
- LLM emits an empty plan → calls tool → reads result → calls another tool → reads → decides which final action to emit = tool-using agent.

The structural test: **could the LLM do its job correctly without calling any tool?** If yes, it's a chained prompt. If no — if you've designed the agent's information access so tools are the only path to ground truth — it's genuinely tool-using.

---

## 5. n8n MCP server limitations (won't fix, work around)

### `removeConnection` can't target `ai_tool` / `ai_outputParser`

The `n8n_update_partial_workflow.removeConnection` operation only inspects `main` connections. It will report "No connections found" even when an `ai_tool` connection clearly exists in the JSON.

**Workaround**: use `n8n_update_full_workflow` to overwrite the connections object wholesale.

### Validator rejects `executeWorkflowTrigger`-only workflows being saved with `active:true`

If you try to save a workflow whose only trigger is an `Execute Workflow Trigger` node while it's `active`, the n8n-mcp validator rejects with: *"Cannot activate workflow with only Execute Workflow Trigger nodes."* This is a false positive — n8n itself accepts this configuration, and you can prove it by looking at any active tool sub-workflow.

**Workaround**: deactivate via UI, patch via API, reactivate via UI. Or just paste the code change directly into the n8n UI and bypass the API entirely.

### `n8n_update_full_workflow` requires `name` in the request body

Even when you're only changing nodes/connections, the API errors with `request/body must have required property 'name'` if you omit `name`.

**Workaround**: always include `name` in the request, even if it's the same as the existing name.

### `n8n_test_workflow` cannot fire Manual Trigger workflows

The MCP tool only supports webhook, form, and chat triggers. Manual Triggers must be fired by a human clicking "Execute Workflow" in the n8n UI.

**Workaround**: ask the user to click the button. Or add a webhook trigger in parallel for programmatic execution.

### `lmChatAnthropic` and `agent` nodes do NOT expose `tool_choice`

You cannot force the model to use a tool from inside n8n. Prompt-only enforcement is the only lever. (See error #2 above.)

### `splitInBatches` two-output topology can collapse during structure normalization

When you remove a node connected to `splitInBatches`, the auto-sanitization sometimes collapses the empty "done" output. This breaks the loop topology silently.

**Workaround**: after any structural change, verify `loop_over_items.main` has both `[]` (done) and `[{node: ...}]` (process) entries. If not, add the missing connection with explicit `sourceIndex: 1`.

---

## 6. AI-fillable schema fields not auto-populated

### Symptom (this run)

`tool_schema_validator` returned `{valid: false, errors: ["candidate_payload missing or not an object"]}`. n8n shows: *"No parameters are set up to be filled by AI. Click on the ✨ button next to a parameter to allow AI to set its value."*

### Why it happens

In the tool sub-workflow node's `workflowInputs.schema`, defining a field with `defaultMatch: false` does NOT make it AI-fillable. n8n requires an additional flag (the one toggled by the ✨ button in the UI) to mark a parameter as something the LLM should populate.

### Fix

Three things must be set together on the toolWorkflow node for AI-fillable schema fields to actually be populated by the agent:

1. **`typeVersion: 2.2`** on the tool node itself (older `typeVersion: 2` does not support the AI-fill flow correctly).
2. **`workflowInputs.matchingColumns`** must list the field IDs the agent should fill, e.g. `["candidate_payload", "abstain_mode"]`. An empty array `[]` is the silent killer — n8n shows "No parameters are set up to be filled by AI" and the agent passes nothing.
3. **`removed: false`** on each schema entry. Looks redundant but it's part of the working configuration.

### Reference (Vitagel production tool, known working)

```json
{
  "parameters": {
    "description": "...",
    "workflowId": {...},
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {},
      "matchingColumns": ["query"],
      "schema": [
        {"id": "query", "displayName": "query", "required": false,
         "defaultMatch": false, "display": true, "canBeUsedToMatch": true,
         "type": "string", "removed": false}
      ],
      "attemptToConvertTypes": false,
      "convertFieldsToString": false
    }
  },
  "type": "@n8n/n8n-nodes-langchain.toolWorkflow",
  "typeVersion": 2.2
}
```

### How we found it

The error message in the n8n UI ("No parameters are set up to be filled by AI. Click on the ✨ button next to a parameter") is a strong signal that the issue is `matchingColumns` being empty. The ✨ icon toggles the field's membership in `matchingColumns`, not any other JSON property. Comparing the workflow JSON before/after clicking ✨ in a known-good Vitagel tool would have shown this directly. We found it by reading a working production tool node and diffing the JSON.

### Lesson

When n8n's UI surfaces a help message about "AI-fillable" parameters, **the `matchingColumns` array on `workflowInputs` is the field to inspect**. It is not enough to set `display: true` and `canBeUsedToMatch: true` on the schema entry — `matchingColumns` must explicitly list the field's `id`. Also, **always set `typeVersion: 2.2`** on `@n8n/n8n-nodes-langchain.toolWorkflow` nodes; the older 2.0 lacks the AI-fill flow.

### Update — `matchingColumns` was not the right answer either

After patching `matchingColumns + typeVersion 2.2 + schema with removed:false`, the agent still failed to populate parameters on re-run. The `candidate_payload` field showed a red expression-error indicator in the UI, meaning n8n was looking for an actual expression in `value` and the empty value broke it.

**The canonical n8n mechanism is the `$fromAI()` expression**, [documented here](https://docs.n8n.io/advanced-ai/examples/using-the-fromai-function/). Each AI-fillable parameter must have an expression in `workflowInputs.value` like:

```
"value": {
  "candidate_payload": "={{ $fromAI('candidate_payload', 'description for the agent', 'json') }}",
  "abstain_mode": "={{ $fromAI('abstain_mode', 'description', 'boolean') }}"
}
```

Function signature: `$fromAI(key, description?, type?, defaultValue?)`. Types: `string` (default), `number`, `boolean`, `json` (for objects and arrays).

Pre-bound (non-AI) parameters use a regular n8n expression in the same `value` map:

```
"value": {
  "utterances": "={{ $('extraction_post_normalizer').first().json.utterances }}"
}
```

So the `value` map mixes upstream-bound expressions (deterministic) with `$fromAI` expressions (agent-bound) for the same tool node. **This is the actual fix.** `matchingColumns` is for upsert deduplication, not AI-filling — that was a misread of the schema metadata.

### Updated lesson

**Use `$fromAI()` in `workflowInputs.value` for every parameter the agent should populate.** Use plain `={{ $('UpstreamNode').first().json.field }}` expressions for parameters pre-bound from upstream. They share the same `value` object. The `schema`, `matchingColumns`, and `typeVersion: 2.2` settings are necessary scaffolding but **none of them substitute for the `$fromAI` expression in `value`**.

Diagnostic shortcut: if you see a red `1🔴` indicator in the n8n UI on a parameter's value field, that's a broken expression — usually because `value` is empty or has the wrong shape.

### Update — exact n8n-generated `$fromAI` syntax includes a comment marker

We toggled the ✨ button manually in the n8n UI and read the JSON to find the canonical format. n8n generates this exact string for an AI-fillable parameter:

```
={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('PARAM_NAME', ``, 'TYPE') }}
```

Three notes:

1. **The `/*n8n-auto-generated-fromAI-override*/` comment is what tells the n8n UI that this expression is from the ✨ button.** Without it, the UI treats the expression as a user-defined static expression, even though it functionally works as a `$fromAI` call. With the comment, the UI renders the parameter as "Defined automatically by the model" (the toggle-on state).
2. **The description argument is empty backticks** (`` ` ` ``), not an empty string `""`. Probably interchangeable, but matching n8n's exact emit avoids any ambiguity.
3. **Other parameters in the same `value` map can be plain expressions** (e.g., `={{ $('UpstreamNode').first().json.field }}`). They co-exist with `$fromAI` expressions in the same object.

### Gotcha — toggling the ✨ on a boolean field can statically set it instead of AI-filling it

When we manually toggled the ✨ button on a `boolean` field, n8n set the value to a **static `true`** (the toggle position) rather than wrapping it in `$fromAI`. The UI's green pill display next to a boolean parameter looks like an "AI-fillable" toggle but is actually a "set static value to true/false" toggle. The way to make a boolean AI-fillable is to use the ✨ icon (separate from the green pill) to wrap the field in `$fromAI('name', ``, 'boolean')`.

Always verify by re-reading the JSON after a UI change. The static-vs-AI distinction isn't visually obvious in the n8n UI for boolean fields.

### Updated lesson (final)

The canonical, working format for AI-fillable parameters in `@n8n/n8n-nodes-langchain.toolWorkflow` (typeVersion 2.2):

```json
{
  "parameters": {
    "workflowInputs": {
      "mappingMode": "defineBelow",
      "value": {
        "ai_filled_param": "={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('ai_filled_param', ``, 'json') }}",
        "upstream_bound_param": "={{ $('UpstreamNode').first().json.field }}"
      },
      "matchingColumns": ["ai_filled_param"],
      "schema": [
        {"id": "ai_filled_param", "displayName": "ai_filled_param", "required": false, "defaultMatch": false, "display": true, "canBeUsedToMatch": true, "type": "object", "removed": false}
      ]
    }
  },
  "typeVersion": 2.2
}
```

Mix `$fromAI` expressions (agent-fillable) with regular `={{ $('node').first().json.field }}` expressions (upstream-bound) freely in the same `value` map. `matchingColumns` lists only the agent-fillable param IDs. Each schema entry needs `removed: false`.

---

---

## 7. n8n's executions API misreports ai_tool sub-execution counts

### Symptom

After a workflow run where the LangChain agent definitely invoked tools (visible in the n8n UI as green checkmarks and "1 item" badges under each tool node), the n8n executions API returned `itemsInput: 0, itemsOutput: 0` for those same tool nodes. Inside a Code node, `$('tool_quote_grounding').all().length` returns 0 even when the tool actually fired.

The same execution had `duration: 2450ms` reported, even though the LLM calls inside the agent took 5-10 seconds each — the duration metadata appears to report only the synchronous orchestration portion, not the full wall-clock including LLM round trips.

### Why it happens

When a tool is wired to a LangChain agent via an `ai_tool` connection, the tool's invocation is a **sub-execution inside the agent's loop**, not a child of the main flow. n8n's executions API and `$('node').all()` accessor only see main-flow nodes. The sub-executions don't surface there.

This is a known n8n quirk for LangChain agent + ai_tool topology — the n8n UI shows the tool fired, but the execution-tree API doesn't. Anything you write that depends on counting tool invocations from the main flow gets a false zero.

### The wrong fix (what we tried first)

We wrote a `tools_invoked` quality check using:
```javascript
let toolInvocations = 0;
try { toolInvocations += $('tool_quote_grounding').all().length; } catch (e) {}
try { toolInvocations += $('tool_schema_validator').all().length; } catch (e) {}
try { toolInvocations += $('tool_uncertainty_check').all().length; } catch (e) {}
```

This always returned 0 even when tools fired. It made our quality_score show 4/5 when it should have been 5/5, and it made us think the agent was fabricating tool calls when it wasn't.

### The correct fix

Read **the agent's verdict for evidence of tool invocation**, not the n8n node graph. The agent's verdict includes a `tool_summary` object that echoes what each tool returned. If the verdict has `tool_summary.quote_grounding.all_grounded` (a boolean), we know quote_grounding ran.

```javascript
let toolSummaryEvidence = 0;
if (verdict.tool_summary?.quote_grounding && typeof verdict.tool_summary.quote_grounding.all_grounded === 'boolean') toolSummaryEvidence++;
if (verdict.tool_summary?.schema && typeof verdict.tool_summary.schema.valid === 'boolean') toolSummaryEvidence++;
if (verdict.tool_summary?.uncertainty && typeof verdict.tool_summary.uncertainty.signal_strength === 'string') toolSummaryEvidence++;
checks.tools_invoked = toolSummaryEvidence >= 2;
```

Caveat: this trusts the agent's self-report. The agent could fabricate a `tool_summary` without calling tools. To detect fabrication, you'd need to cross-check by inspecting (a) whether the values in `tool_summary` are plausible given the input, OR (b) running an after-the-fact deterministic verification that compares.

### Lessons

- **Don't trust `$('node').all()` to detect tool invocations** when the tool is wired via `ai_tool`. The n8n graph doesn't see ai_tool sub-executions as main-flow nodes.
- **Don't trust `duration` metadata** as a proxy for wall-clock time on workflows with LangChain agent loops. The number is too small to be real and misleads diagnostic reads. Use the n8n UI's execution timeline instead.
- **Reading the agent's own `tool_summary`** is the practical way to score whether the agent did its job, even if it shifts the trust boundary onto the agent.

---

## 8. Code node silently dropped items because of "Run Once for All Items" mode

### Symptom

`text_pre_processor` (Code node) correctly emitted **2 items** (one per transcript). The downstream `transcript_reader` (also a Code node) emitted only **1 item**. The `loop_over_items` (`splitInBatches`) then iterated only once, processing OPP-001 but completely skipping OPP-002. The execution finished with `final_aggregator` reporting `"1 update(s), 0 abstain(s)"` — looking superficially correct.

In the n8n editor, the visible item-count badges showed:
- `text_pre_processor`: 2 items
- `transcript_reader`: 1 item
- `loop_over_items`: 1 item, 2 executions visible (✓2)

The user spotted this from the editor's flow visualization, not from the API output.

### Why it happened

Code nodes in n8n have two execution modes set via `parameters.mode`:

- **`runOnceForAllItems`** (default in many cases): the JS runs **one time total**. `$input.first()` returns the first input item; the rest are inaccessible unless you iterate `$input.all()`.
- **`runOnceForEachItem`**: the JS runs **once per input item**. `$json` is the current item; `$input.first()` is not how you access it.

Our `transcript_reader` was in `runOnceForAllItems` mode and used `$input.first().json` to access the input — so it processed only OPP-001 and discarded OPP-002.

### The fix

Switch to per-item mode and rewrite the access pattern:

```json
"parameters": {
  "mode": "runOnceForEachItem",
  "jsCode": "const body = $json.body ?? $json; ... return { json: output };"
}
```

In per-item mode:
- `$json` is the current item's data.
- Return shape is `{ json: ... }` (a single object), not `[{ json: ... }]` (an array).
- The runtime calls the code N times, once per input item.

### Lessons

- **Always check the Code node's `mode` parameter** when a downstream node receives fewer items than expected. The default isn't always right for transformation steps that should process every item.
- **Match the access pattern to the mode**: `$input.first()` / `return [{json:...}]` for `runOnceForAllItems`; `$json` / `return {json:...}` for `runOnceForEachItem`.
- **Trust the n8n UI's item-count badges over API metadata** — the UI showed 2→1→1, which immediately localized the dropoff to `transcript_reader`. The API returned 200 OK and `executedNodes: 12` with no error.
- **Single-OPP runs that "succeed" silently** are the most dangerous failure mode in n8n loops, because the workflow appears correct end-to-end but is dropping work on the floor.

---

## 9. `$('NodeName').first()` returns the FIRST item, not the current loop iteration

### Symptom

After fixing #8 (transcript_reader producing 2 items via per-item mode), the loop now iterated twice as intended. But OPP-002 produced wrong / mismatched data downstream — the agent drafted for OPP-002 but the post-normalizer attached OPP-001's utterances and `_test_meta`. `tool_quote_grounding` then errored because it was matching OPP-002's evidence quotes against OPP-001's utterances.

### Why it happened

Inside a `splitInBatches` loop, downstream Code nodes that need to access *upstream* data (data from before the loop) often use `$('SomeUpstreamNode').first().json`. This worked when the upstream node only had one item in its output. But when an upstream node outputs an array of items (like `transcript_reader` did after fix #8), `.first()` **always returns the first array element regardless of which loop iteration is currently running**.

So in iteration 2 (OPP-002), the agent's input came from the loop and was correctly OPP-002, but the post-normalizer's `$('transcript_reader').first()` still returned OPP-001's data. Result: silent data mismatch — wrong utterances paired with right draft.

### The fix

Read from the loop node itself, not the upstream-of-loop node. `$('loop_over_items').first().json` returns the **current iteration's item** because `splitInBatches` re-emits one item at a time:

```javascript
// WRONG: returns OPP-001 forever, even on iteration 2
const tr = $('transcript_reader').first().json;

// RIGHT: returns the current iteration's item
const loopJson = $('loop_over_items').first().json;
const utterances = loopJson.utterances;
const opportunity_id = loopJson.opportunity_id;
```

Alternative: pass the data through the loop in the regular flow (so `$input.first().json` carries it forward). But reading from the loop node directly is cleaner when the upstream-of-loop data is large or many fields wide.

### Lessons

- **Inside a `splitInBatches` loop, reach back to the LOOP node**, not nodes upstream of it, to access per-iteration data. The loop's output is always the current item; an upstream multi-item node's `.first()` is always the first item.
- **This bug compounds with #8**: when an upstream Code node moves from single-item to multi-item output, every downstream `$('UpstreamNode').first()` reference becomes a per-iteration mismatch waiting to happen. After any mode change on a Code node, audit downstream `.first()` references.
- **Diagnostic test**: in a multi-OPP run, log `opportunity_id` at every Code node. The first OPP where the logged ID stops matching the iteration is where the cross-talk starts.

---

## 10. n8n's tool schema validator rejects empty arrays — agent must skip the tool call entirely

### Symptom

When the review_agent processed Transcript 2 (thin transcript / abstain case), it correctly called `quote_grounding` with `evidence_list: []` (an empty array — there are no claims to ground in an abstain). The tool node errored out:

```
Received tool input did not match expected schema
✖ Value must be a non-empty object or a non-empty array
  → at evidence_list
```

The sub-workflow itself is designed to handle the empty-array case (returns `{all_grounded: true, summary: 'no_claims'}`). But n8n's `toolWorkflow` node validates the input schema **before** the sub-workflow runs, and rejects empty arrays at the schema layer.

### Why it happens

This is a [known n8n bug](https://community.n8n.io/t/ai-tool-schema-doesnt-permit-arrays/82930) in the `@n8n/n8n-nodes-langchain.toolWorkflow` node. When you declare a parameter as `type: array`, n8n's input validator enforces `minItems: 1` implicitly — it doesn't trust empty arrays as valid agent-fillable inputs. There's no schema config flag exposed to override this.

### The fix

Since we can't loosen the validator, **change the agent's behavior** instead. Update the system prompt to instruct: when `draft_abstain` is true (the only case where evidence_list would be empty), **skip the `quote_grounding` call entirely**. There's nothing to ground in an abstain anyway, so the call is wasted.

Example branching in the system prompt:

```
## If draft_abstain is TRUE:
1. DO NOT call quote_grounding. Skip it.
2. Call schema_validator(... abstain_mode=true) to verify the abstain shape.
3. Call uncertainty_check to confirm the abstain decision.
4. Approve the abstain or revise.
```

This is also better agent behavior architecturally — the agent now branches its tool-calling pattern based on the input state, which is exactly the "tool-using agent that decides things" property the take-home brief is asking for.

### Alternative we considered

We could tell the agent to pass a placeholder like `[{field:'_abstain_placeholder',quote:'',timestamp:''}]` to bypass the empty-array check, but that's a hack and would make `quote_grounding`'s output meaningless. Skipping the call is cleaner.

### Lessons

- **n8n's toolWorkflow validator rejects empty arrays**, even when your sub-workflow handles them correctly. Don't depend on empty-array tool calls.
- **Branched tool-calling patterns** (different tools called based on input state) are not just acceptable — they're the cleanest answer to "the agent decides things". Encode the branch in the system prompt explicitly.
- **Wasted tool calls are an architectural smell.** If a tool would have nothing to do given the current state, the agent should know not to call it.

---

## 11. Extraction agent emits "grounded" quotes that don't actually contain the claimed value

### Symptom

After a successful run with `quote_grounding.all_grounded: true`, manual review of the reasoning_trace surfaced two issues:

1. **Champion citation didn't ground the claim**: agent claimed `champion: "Priya"` with quote `"You'll loop in Ben for next Tuesday?"` (timestamp [02:05]). The quote doesn't contain "Priya" or any evidence Priya is the champion. The literal substring match in `quote_grounding` succeeded (the quote IS in the transcript at that timestamp) but the **semantic** grounding is wrong.

2. **Close-date year ambiguity**: agent picked `close_date: "2027-10-15"` when the transcript says *"Actually pushing to early Q4 now. Our 2027 portfolio review moved."* The "2027" mention is about the portfolio review (a separate event), not the deal close. The agent conflated the two and pushed the deal year forward by one.

### Why it happens

The `quote_grounding` tool we built does a **substring match** between the cited quote and the source utterance — that's how it confirmed `all_grounded: true`. But substring match doesn't verify that the cited quote actually justifies the claimed value. The agent can pick *any* quote from the right timestamp and the grounding tool will accept it.

This is a classic citation-quality failure: **proximity vs. relevance**. The model picks contextually-near quotes that aren't actually evidence for the claim.

### The fix (prompt-side)

Add explicit guidance in the extraction agent's system prompt:

```
# CRITICAL: QUOTE GROUNDING
The `quote` you cite MUST contain the actual VALUE you are claiming, not just adjacent context.
- BAD: champion="Priya" with quote "You'll loop in Ben for next Tuesday?" — quote doesn't contain "Priya".
- GOOD: champion="Priya" with quote "Thanks for the time, Priya" or a Priya utterance.
Before writing each reasoning_trace entry, verify the quote literally contains the claimed value.

# CRITICAL: DATE INTERPRETATION
When transcript references a quarter or year ambiguously, interpret as the NEAREST FUTURE date.
- "Q4" alone → next Q4 (Q4 2026).
- A mention of a different year (e.g., "2027 portfolio review") is context, NOT necessarily the close date.
```

Bad/good examples in the prompt are critical — the model copies its citation pattern from the example more than from rules.

### The fix (tool-side, deferred)

Long-term, `quote_grounding` should also do a **value-presence check**: does the cited quote contain (or strongly imply) the claimed value? This is harder than substring match — it requires either an LLM-as-judge step or a heuristic (numeric values: regex; named entities: substring with case folding; dates: parse). Out of scope for this iteration; noted in the "what I'd add with 4 more hours" list.

### Lessons

- **Substring grounding catches hallucinated quotes but not bad citations.** A quote being IN the transcript doesn't mean it's the right quote for the claim.
- **Few-shot examples in the prompt are more effective than rules** for this kind of citation-quality issue. The model imitates good examples; abstract rules slip.
- **For ambiguous date references, give the model an explicit nearest-future heuristic.** Without it, the model defaults to whatever year is mentioned anywhere in the transcript, even if that year refers to a different event.
- **Inspect grounded outputs manually before declaring victory.** "all_grounded: true" is necessary but not sufficient.

---

## Tier list of lessons

**S-tier (don't ever forget):**
- Disable agent retries before doing anything else when you see ITPM errors.
- ITPM is rolling 60s, summed across all calls.
- Different models have different ITPM ceilings on the same tier.
- "tool-using agent" ≠ "chained prompt" — the LLM must read tool results and branch on them.
- Don't combine `structuredOutputParser` with `ai_tool` connections.

**A-tier (high-leverage):**
- Pre-bind heavy inputs in tool nodes. The agent only fills small fields.
- Slim tool return payloads. Echoed source data is the silent ITPM killer.
- Haiku tool-call adherence is weaker than Sonnet's in n8n's wrapper.
- n8n doesn't expose `tool_choice`. Plan around it.

**B-tier (good to know):**
- `n8n_update_full_workflow` requires `name`.
- `splitInBatches` two-output topology is fragile under partial updates.
- `removeConnection` only sees `main` connections.

---

## Process lesson (meta)

We spent ~5 hours on this with significant back-and-forth. In retrospect, two things would have saved the most time:

1. **Read the API docs for ITPM behavior up front** — specifically what counts toward the limit and over what window. We thought we understood it; we didn't. The "rolling 60s sum across all calls" was the missing piece.
2. **Pick the architecture that satisfies the criterion before optimizing for reliability.** We optimized away the agent's tool-calling because Haiku was unreliable, then had to rebuild it once we realized the criterion required it. The reliability problem was real but the right answer was "use a different model for that step," not "remove the agent loop."
