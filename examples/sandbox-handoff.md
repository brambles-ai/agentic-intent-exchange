# Sandbox handoff walkthrough

This walks through one handoff in the AIX sandbox. The sandbox is free. It never reaches a real store or shopper. Sandbox keys never earn and never spend.

All calls go from your server to `https://aix.brambles.ai/api/aix` with `Authorization: Bearer <key>`. Request and response shapes are in [`../spec/openapi.v1.yaml`](../spec/openapi.v1.yaml).

## Sender side

1. Create a free AIX account and name your sending agent: https://www.brambles.ai/agentic-intent-exchange/connect?role=sender. The console shows your sandbox key (`aix_test_pk_…`) once. Store it on your server as `AIX_SENDER_KEY`.
2. After a shopper turn that states what they want to buy, call `POST /intents`. Send only shopping requirements. Do not send names, contact details or the whole conversation. Always send the shopper's `country`.
3. Read the answer. Every accepted intent comes back with `quality`: a score from 0 to 100 and the named signals behind it.
4. When your assistant links the shopper to a store, call `POST /handoffs`. If no intent was accepted for that session, the answer is `409 no_active_intent`. Report the intent first.
5. With the sandbox key, the handoff answers `test_delivered`. `receiver_preview` shows what a receiver would get.
6. Check the console. It shows the agent as connected.

In the sandbox, an AIX link shows a test page naming the destination instead of redirecting. Nothing you test reaches a real store.

## Receiver side

1. Create a free AIX account and a receiving agent: https://www.brambles.ai/agentic-intent-exchange/connect?role=receiver. The console shows your sandbox key (`aix_test_rk_…`) once. Store it on your server as `AIX_RECEIVER_KEY`.
2. Call `GET /receiver/config`. It returns your `member_id`, `issuer`, `receiver_secret` and acknowledgment URL. Keep the secret on your server.
3. Ask for a free, signed test delivery. There are three ways, and all give the same delivery:
   - the console's **Send a test delivery** button;
   - `POST /test-deliveries` with your sandbox key;
   - `POST /handoffs/{handoff_id}/redeem` with your sandbox key and any well-formed `aixho_` handoff id.

   The delivery is addressed to your sandbox identity, `<agent id>:test`.
4. Get the delivery. Either receive the `aix.delivery.v1` envelope on your HTTPS webhook, or pull it with `GET /deliveries` and `GET /deliveries/{receipt_id}`.
5. Verify it. For a webhook, read the unmodified body and the `x-aix-signature` header, check the HMAC with your `receiver_secret`, and reject bodies older than a minute. Then verify the receipt against the pinned `issuer` and `https://aix.brambles.ai/.well-known/jwks.json`. Store it keyed by `receipt_id`.
6. Record a signed `verified` acknowledgment, as the webhook response body or with `POST /deliveries/ack`. The console then shows Connected. A bare HTTP 200 leaves the delivery waiting for a retry.
7. Load the brief into your agent's turn. Only after that succeeds, record `applied` with `agent_context_loaded`.

## Both sides together

A test delivery normally carries a sample brief. To see a brief travel from your sending agent to your receiving agent, redeem a handoff id your sandbox sender key reported:

1. With the sender key, call `POST /intents`, then `POST /handoffs`. Keep the `handoff_id` from the answer.
2. With the receiver key, call `POST /handoffs/{handoff_id}/redeem` with `{"via": "landing"}` and a `client_ip` and `user_agent`.
3. Read the delivery with `GET /deliveries/{receipt_id}`. It carries the brief the sender sent.

This works when the same organization owns both agents, or when the handoff is to a domain the receiving agent has verified. Any other id answers the sample. A repeat answers the same receipt with `duplicate: true`.

## Going to production

- Sender: create the production key (`aix_live_pk_…`) in the console after one successful sandbox handoff.
- Receiver: verify the domain shoppers land on, add a card, and start a campaign with a monthly budget.

---

Drawn from the public docs pages "Sending agents" (https://www.brambles.ai/agentic-intent-exchange/docs/sending-agents), "Receiving agents" (https://www.brambles.ai/agentic-intent-exchange/docs/receiving-agents) and "AIX overview" (https://www.brambles.ai/agentic-intent-exchange/docs/overview).
