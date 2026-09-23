# market-landscape-agent

A **LangChain agent** that maps a market category and sizes it bottom-up, using
**Nimble** for web data. Give it a category and inclusion criteria; it discovers
candidate vendors, researches each one, rules every candidate include or exclude with a
reason, and computes a bottom-up size estimate from the included set, checking it
against any published top-down estimates it finds and flagging real disagreement rather
than averaging over it.

| File | Pattern | Who runs the research loop |
| --- | --- | --- |
| `agent.py` / `run.py` | **A** - LangChain + Nimble Search API | the LangChain agent |
| `agent_api_v2.py` | **B** - Nimble Web Search Agent (Agent API) | Nimble |

Both patterns produce the same `LandscapeExtractionBatch` shape, so
`landscape_model.build_result` (dedupe, rule enforcement, bottom-up sizing,
reconciliation) is identical downstream. Only the retrieval step changes.

## Why the sizing is not the LLM's job

Asked to just "estimate the market size," an LLM will produce a single confident number.
`landscape_model.py` does three things in plain Python instead:

- **Dedupe and enforce the include/exclude rule in code**, not just trust it: a
  candidate marked `include=True` with no evidence URL is forced to excluded; a
  candidate marked excluded with no reason gets a generic one rather than silently
  passing through unexplained.
- **Bottom-up estimate as a range, not a point**: sums disclosed ARR where a vendor
  states one, and separately estimates ARR for funded vendors without a disclosed
  figure using a stated heuristic (25% of total funding raised, a rough SaaS rule of
  thumb, not a measurement). The gap between the two bounds, plus a confidence grade
  based on how many vendors actually disclosed a figure, is the honest picture.
- **Reconciliation against published estimates**: parses the dollar figure out of each
  published estimate's text and flags it when it sits outside 2x-0.5x of the bottom-up
  midpoint, rather than averaging bottom-up and top-down into one number.

## Run

```bash
uv sync
cp .env.example .env       # NIMBLE_API_KEY + an LLM_MODEL and its key

uv run python run.py "LLM observability and evaluation platforms" \
  --criteria "dedicated LLM observability or evaluation product, not a general-purpose APM vendor" \
  --criteria "self-serve or clearly documented deployment" \
  --count 30

uv run python agent_api_v2.py "LLM observability and evaluation platforms" \
  --criteria "dedicated LLM observability or evaluation product" --count 30
```

`--criteria` is repeatable, one inclusion criterion per flag. `LLM_MODEL` is
provider-agnostic via `init_chat_model`: `openai:gpt-5.1` (default),
`anthropic:claude-sonnet-5`, ... Install the matching provider package.

## Files

- `schema.py` - `VendorCandidate`, `PublishedEstimate`, `BottomUpEstimate`,
  `ReconciliationFlag`, `LandscapeResult`.
- `landscape_model.py` - dedupe, rule enforcement, the bottom-up cost model, and
  reconciliation. No LLM calls.
- `config.py` - the discovery/research agent's role and system-prompt builder.
- `agent.py` - the Pattern A LangChain agent (broaden with snippets, narrow with
  full-content per vendor domain) and its `nimble_search` tool.
- `agent_api_v2.py` - the Pattern B driver.
- `run.py` - Pattern A CLI and the shared artifact writer.

## Example

`examples/` - a real run on "LLM observability and evaluation platforms": 13 vendors
included (LangSmith, Langfuse, Arize AI, Braintrust, Patronus AI, Galileo AI, Helicone,
Fiddler AI, and others), 12 candidates excluded with specific reasons (general APM
vendors like Datadog, Honeycomb, and Elastic Observability correctly separated from
dedicated LLM-observability products, a parked domain flagged rather than assumed
active, duplicate/gateway-only entries caught), and a bottom-up size estimate ($0 to
$20.75M, confidence: low, since almost none of these early-stage vendors disclose
revenue).

The reconciliation step found a real published figure, MarketsandMarkets' "Observability
Tools and Platforms" market at $11.91B, and correctly flagged it as wildly larger than
the bottom-up estimate (over 1000x). That is not a bug: the published figure sizes the
entire parent observability category (APM, logs, infra monitoring, and more), not the
narrow LLM-observability sub-segment this run mapped, and no public report sizes that
sub-segment specifically yet. The gap itself is the finding: it shows how early and
under-measured this specific niche still is relative to its parent category, which is
exactly the kind of disagreement `landscape_model.reconcile` is built to surface rather
than paper over.
