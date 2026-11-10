# Handoff

A handoff moves one shopper's brief from a sender to the one receiver the shopper is going to. The handoff id is the reference that travels with the shopper. It starts with `aixho_`.

## Steps

1. A shopper talks to a participating sender. The sender reports the current shopping task, or AIX extracts it, as a versioned, structured brief. AIX scores its quality before anything is delivered.
2. The sender reports the handoff to a store. If the receiver has verified that domain and runs a live campaign that matches the task and its quality bar, AIX answers an AIX link and a parameter for the sender's own link. Nothing is charged yet.
3. The shopper opens the link or lands on the store with the parameter within an hour, or their agent completes an order through the store's API and the receiver redeems the handoff id. Brambles' checks confirm it is a real visit. Then the recipient-bound receipt is issued, the receiver is charged and the sender is credited. This is the delivered intent.
4. The receiver obtains the receipt through an HTTPS webhook, or retrieves its exact delivery through the authenticated API or MCP integration.
5. The receiver's server verifies the receipt and records a signed `verified` acknowledgment after durable storage. A plain HTTP 200 does not confirm verification.
6. The receiver loads the brief into its agent turn or applies it to catalog candidates. Only after that action succeeds does its server record `applied`.
7. The receiver may report an observed outcome against that exact receipt. An outcome report is not proof of conversion lift.

## Rules for the handoff id

- A handoff a receiver would pay for answers the status `delivers_on_click`.
- Each handoff id is delivered once, within one hour of the handoff report, and only on the receiver's verified domains.
- Only the store a handoff was for can redeem it.
- Nothing is charged when a sender merely reports a handoff.
- Automated opens never count. Crawlers, link previews, prefetch, HTTP libraries and headless browsers pass through to the store without a delivery.
- A shopper whose browser sends Global Privacy Control reaches the store without their brief. Nothing is delivered or charged.

## Ways a receiver can receive

The first four are ways a delivered intent is confirmed. The last three are ways the receiver gets the receipt.

### Click

The sender shows `handoff_url`, an AIX link (`https://aix.brambles.ai/h/…`). It delivers on the shopper's click and lands on the store page. The shopper arrives with `?aix_receipt=<receipt_id>`.

### Landing

The sender keeps its own link and adds `aix_param` (`aix=aixho_…`) to the store URL inside it. The shopper arrives with `?aix=<handoff_id>`. The receiver picks it up in one of three ways:

- The AIX tag, one line on every page: `<script src="https://aix.brambles.ai/aix-tag.js" async></script>`. It claims the handoff and fires an `aix:arrival` event with the receipt id.
- The receiver's own server: `POST /handoffs/{handoff_id}/redeem` with `{"via": "landing"}` and the landing request's `client_ip` and `user_agent`. AIX runs the checks a click runs, so crawlers, link previews and prefetches are never charged.
- A Brambles shopping assistant on the store, which already does this.

### Agentic checkout

When a shopper's agent buys through the store's checkout API, nobody visits a page. The handoff id travels in the checkout:

- UCP: `"attribution": { "aix_handoff": "aixho_…" }`
- ACP: `"affiliate_attribution": { "provider": "aix.brambles.ai", "token": "aixho_…", "touchpoint": "first" }`

When the checkout completes an order, the receiver's server redeems it with `POST /handoffs/{handoff_id}/redeem` and `{"via": "checkout", "order_id": "<your order id>"}`. A cart that never becomes an order is never charged. AIX adds no checkout of its own and never touches the payment.

### Personal agent

A store lists `brambles.ai/aix` in the `extensions` of its `/.well-known/poppy.json`. The shopper's agent puts the handoff id in the `context` of its first message to the store's agent. When the conversation completes an order, the receiver redeems it with `{"via": "agent", "order_id": "<your order id>"}`. A conversation that ends without an order is never charged. The Personal Agent Protocol is a draft (0.3) and can change.

### Webhook

AIX pushes each delivery to the receiver's HTTPS webhook as an `aix.delivery.v1` envelope, signed in the `x-ain-signature` header. The receiver answers with its signed `verified` acknowledgment as the response body. Without one, the push is retried for about 40 hours: 12 tries, from 5 minutes up to 6 hours apart.

### Polling

`GET /deliveries` and `GET /deliveries/{receipt_id}` pull deliveries without a webhook. One delivery returns the verified brief and shopping context.

### MCP

The endpoint is `https://aix.brambles.ai/api/aix/mcp`, over Streamable HTTP, with `Authorization: Bearer <key>`. A key sees only the tools it can call. Receiver tools include `subscribe_intents` (a poll, not a push stream), `get_intent_context`, `redeem_handoff`, `acknowledge_delivery`, `delivery_status`, `report_outcome` and `get_activity`.

---

Drawn from the public docs pages "AIX overview" (https://www.brambles.ai/agentic-intent-exchange/docs/overview), "Sending agents" (https://www.brambles.ai/agentic-intent-exchange/docs/sending-agents), "Receiving agents" (https://www.brambles.ai/agentic-intent-exchange/docs/receiving-agents), "Protocols" (https://www.brambles.ai/agentic-intent-exchange/docs/protocols) and "Personal agents" (https://www.brambles.ai/agentic-intent-exchange/docs/personal-agents).
