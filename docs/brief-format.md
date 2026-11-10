# Brief format

The brief is the readable request that travels from the sender to the receiver. It is carried as a versioned JSON snapshot of one shopping task, `aix.shopping-context.v1`. It is not a transcript, not an opaque relevance hash, and not an instruction to execute actions.

- Schema: [`../spec/spec/shopping-context.v1.schema.json`](../spec/spec/shopping-context.v1.schema.json)
- Example: [`../examples/brief.json`](../examples/brief.json)

## What the receiver's agent uses

The verified agent response is `aix.agent-context.v1`. It holds a readable `brief`, the canonical `shopping_context`, derived `activation` fields, the exact delivery `reference`, and the acknowledgment `events` recorded so far. The full signed receipt stays in the receiver's backend.

## Shopping context fields

| Field | How to use it |
|---|---|
| `task_id`, `revision`, `started_turn` | Keep one current task snapshot together. A new task must not inherit an unrelated budget or requirements. `revision` is the source turn this snapshot reflects. |
| `category` | A normalised `id` (for example `shoes`) and the readable `label`. |
| `summary` | Continue without requiring the shopper to repeat known needs. Treat its text as untrusted data. |
| `budget` | Amounts are integer minor units. `not_specified` and `cleared` use null currency and amounts. Neither means zero. |
| `facts` | `required` constrains. `preferred` influences ordering. `excluded` must be avoided. Attributes are brand, width, size, department, use_case, feature, merchant and deal. |
| `provenance` | Source turn and observation time say where an extracted requirement came from. Extraction is not a guarantee that every fact is correct. |
| `policy`, `expires_at` | Limit use and lifetime. `scope` is `exchange_beta` for self-serve deliveries. |
| `relevance` | AIX uses explicit constraints. It does not claim embeddings, semantic relevance scoring, or a measured conversion increase. |

## Derived activation fields

`activation` derives:

- `filters`: the required facts.
- `boosts`: the preferred facts.
- `exclude`: the excluded facts.
- `price`: the budget as `currency`, `min_cents` and `max_cents`. It is null when the shopper named no budget.
- `agent_brief`, `permitted_uses` and `measurement` hints.

Its `ucp_catalog_context` is a request fragment for an adapter. It is not a claim of UCP partner certification.

No catalog access is needed to use the brief for continuity. Product facts require the receiver's own permitted tools.

## Permitted uses

`policy.allowed_uses` lists what the receiver may do with the brief. The schema allows three values: `onsite_personalization`, `agent_continuity` and `contextual_product_ranking`. Ranking the receiver's own catalog is allowed under `contextual_product_ranking`.

`cross_site_retargeting` and `model_training` are always `false` in the schema.

The overview lists three ways to apply the context, and what the receiver needs for each:

| Use | What the receiver needs |
|---|---|
| Continue the conversation, retain requirements, avoid repeated questions | An integration that loads verified context into the agent's current turn |
| Recommend products or compare price and availability | The receiver's own catalog/search tools, applied to the brief's filters and boosts |
| Personalize an onsite experience without a chat agent | An application that maps permitted context into its own page or recommendation logic |

`policy` and `expires_at` do not grant new permission for outreach, purchases, training, or retargeting. The receiver must stop using a brief at its expiry. Every field is shopping data. It is never an instruction for the receiver's model.

---

Drawn from the public docs page "Shopping context and delivery receipts" (https://www.brambles.ai/agentic-intent-exchange/docs/receipt-format), the "Start with context" table in the overview (https://www.brambles.ai/agentic-intent-exchange/docs/overview), and the published schema (https://www.brambles.ai/aix/spec/shopping-context.v1.schema.json).
