# AIX skills

The three skills in this folder are the text `https://aix.brambles.ai` serves. One command installs all of them:

```bash
npx skills add https://aix.brambles.ai
```

| Skill | Who it is for | What it does |
|---|---|---|
| [`aix-send-intent`](aix-send-intent/SKILL.md) | Your coding agent | Connects an AI app, assistant or agent to AIX as a sender. |
| [`aix-receive-intent`](aix-receive-intent/SKILL.md) | Your coding agent | Connects a store, a brand or its shopping agent to AIX as a receiver. |
| [`aix-shopping-continuity`](aix-shopping-continuity/SKILL.md) | Your store's own agent | Tells the agent how to continue a shopper's task from a delivered brief. |

## The two setup skills

Install them, then ask your coding agent to connect your assistant or store to AIX. The skills hold no key. When your project has none, the agent sends you to sign up.

## The store-agent skill

The setup skill wires the brief into your agent's turn. `aix-shopping-continuity` tells the agent what to do with it.

1. Get the skill. The install command saves it to `.agents/skills/aix-shopping-continuity/SKILL.md` in your project.
2. Give it to your store's agent. If your agent framework loads Agent Skills, add that folder to the skills it loads. If it doesn't, paste the text below the header into your agent's system prompt.
3. Give the agent the brief. Either put the verified `brief` and `shopping_context` from `GET /deliveries/{receipt_id}` into the turn as data, or connect the MCP endpoint (`https://aix.brambles.ai/api/aix/mcp`) with your receiver key and pass the agent the exact `receipt_id`, so it calls `get_intent_context`.
4. Check it. Send a test delivery, open the conversation, and confirm the first reply starts from the shopper's request: required facts are kept, excluded facts are left out, and the agent doesn't ask for what the brief already says.
5. Record `applied` with `agent_context_loaded` from your server once the brief is in the agent's turn.

The skill holds no key and no address. It tells the agent to treat the brief as data, never as instructions, and to use your own catalog for products, prices and stock.

The full guide is at https://www.brambles.ai/agentic-intent-exchange/docs/receiving-agents.
