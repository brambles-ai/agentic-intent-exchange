# AIX (Agentic Intent Exchange)

AIX is an exchange operated by Brambles.ai; it is not a protocol and it is not a standard. It carries a shopper's brief from a participating sender (an AI app, assistant or agent) to a participating receiver (a store, a brand or its shopping agent), so the receiver can continue the shopper's task without asking them to repeat it.

The white paper is published at https://www.brambles.ai/agentic-intent-exchange/paper.

## What is in this repository

| Path | Contents |
|---|---|
| `spec/` | The public OpenAPI document and the three public JSON Schemas, unmodified. The layout mirrors `https://www.brambles.ai/aix/`, so the OpenAPI document's relative schema reference resolves. |
| `docs/` | Short notes on the brief, the signed receipt and the handoff, drawn from the public docs. |
| `examples/` | One example brief as JSON, and a walkthrough of a sandbox handoff. |
| `skills/` | The three AIX skills, exactly as `https://aix.brambles.ai` serves them: two setup skills for coding agents, and one for a store's own agent. |
| `paper/` | The AIX white paper, version 1.0, as a PDF. It is also [on the web](https://www.brambles.ai/agentic-intent-exchange/paper). |

## How to try it

The sandbox is free and never reaches a real store or shopper.

1. Create a free account as a [sender](https://www.brambles.ai/agentic-intent-exchange/connect?role=publisher) or as a [receiver](https://www.brambles.ai/agentic-intent-exchange/connect?role=receiver). The console shows your sandbox key once.
2. Add the AIX skills to your coding agent:

   ```bash
   npx skills add https://aix.brambles.ai
   ```

3. Ask the agent to connect your assistant or store to AIX. The installed skills hold no key.
4. Follow [`examples/sandbox-handoff.md`](examples/sandbox-handoff.md).

The public docs are at https://www.brambles.ai/agentic-intent-exchange/docs:

- [Overview](https://www.brambles.ai/agentic-intent-exchange/docs/overview)
- [Sending agents](https://www.brambles.ai/agentic-intent-exchange/docs/sending-agents)
- [Receiving agents](https://www.brambles.ai/agentic-intent-exchange/docs/receiving-agents)
- [Shopping context and delivery receipts](https://www.brambles.ai/agentic-intent-exchange/docs/receipt-format)
- [Protocols](https://www.brambles.ai/agentic-intent-exchange/docs/protocols)
- [Personal agents](https://www.brambles.ai/agentic-intent-exchange/docs/personal-agents)

## Status

Copied from the capability table in the public [overview](https://www.brambles.ai/agentic-intent-exchange/docs/overview).

| Area | Status |
|---|---|
| Self-serve sign-up, sandbox and production keys, AIX console | Live |
| Structured brief scored for quality before delivery, recipient-bound signed receipt, receipt-bound acknowledgments | Live |
| Setup skills (`npx skills add https://aix.brambles.ai`) and MCP for sending and receiving | Live |
| Click-confirmed and landing-confirmed delivery, agentic checkout redemption (UCP, ACP) | Live |
| Receiver campaigns billed per delivered intent through Stripe Billing; sender earnings through Stripe Connect | Live |
| Daily caps for new senders, one-hour link window, rate limits | Live; see the [reference](https://www.brambles.ai/agentic-intent-exchange/docs/reference#limits) |
| Two-way workflows | Broader development build; counterpart validation required |
| AP2, A2A and AdCP | Not built; see [Protocols](https://www.brambles.ai/agentic-intent-exchange/docs/protocols) |
| Wallets, blind relay, multi-hop exchange, full ZK/TEE integration | Advanced development; not beta guarantees |

## Rights

AIX is the subject of a US provisional patent application filed August 7, 2026. No licence has been chosen for these files yet. You may read them and link to them. © Brambles.ai.
