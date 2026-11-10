---
name: aix-send-intent
description: Connect a shopping assistant to the Agentic Intent Exchange (AIX) as a sender so it sends shopping intent and hands shoppers off to stores with that context. Use when asked to connect an assistant to AIX or to send intent to AIX, or when an AIX setup prompt names this skill.
---

# Send shopping intent to AIX

AIX carries a shopper's current shopping task from your assistant to the store
you send them to. Your server reports two events:

1. **Intent**: after a shopper turn that states or changes what they want to buy.
2. **Handoff**: when your assistant sends the shopper to a store link.

Keep your own links. When a participating store would pay for a handoff, AIX
answers with `landing_url`: the store URL with `aix=<handoff id>` appended
(`aix_param`). Use it as the store page inside your affiliate, ad or plain link,
and the store continues from the shopper's task when they land. Otherwise keep
the store URL unchanged.

API base: https://aix.brambles.ai/api/aix

## Rules

- The key is a server secret. Read it from `AIX_SENDER_KEY` in the server
  environment. Never put it in browser or mobile code, logs, commits or a model
  prompt. If it is missing, ask the user to create a free AIX account and sending agent at
  https://www.brambles.ai/agentic-intent-exchange/connect?role=publisher (it shows the key once), then to store it as
  `AIX_SENDER_KEY`. Never invent one.
- Sandbox keys (`aix_test_pk_…`) never reach a real store and never earn. A
  person creates the production key (`aix_live_pk_…`) in the AIX console after
  a test handoff succeeds. Do not switch to production yourself.
- AIX must never slow down or break the shopper's experience. Call it from the
  server with a 3 second timeout and carry on normally if it fails.
- Send only shopping requirements. Never send names, emails, phone numbers,
  addresses, account or order numbers, or the whole conversation.
- Forward privacy signals in `privacy`: Global Privacy Control (`Sec-GPC: 1`)
  as `gpc`, a sale/sharing opt-out as `do_not_sell`, a possible minor as
  `minor`, withheld consent as `consent: "denied"`. AIX then stores nothing
  and withdraws what that session sent earlier.
- Always send `country`, the shopper's two-letter ISO 3166-1 code. Wire it in
  automatically on the server and never ask the shopper: your host's geolocation
  header (Vercel `x-vercel-ip-country`, Cloudflare `CF-IPCountry`, CloudFront
  `CloudFront-Viewer-Country`, App Engine `X-AppEngine-Country`), a GeoIP
  lookup, or on mobile the device region. If none is available, send `locale`
  with a region (`"en-GB"`) and AIX uses its region. Production suppresses
  intents with neither (`country_required`); the sandbox accepts them with a
  `warning`.
- Shoppers in the EEA, the UK or Switzerland are shared only with their consent,
  and their click on a store link is that consent once they have been told what
  it does. Show one line under your assistant to those shoppers: "Clicking a
  store link shares your shopping request with that store." Then report their
  handoffs with `"consent": "granted"` (section 3), also when you report a link
  before it is shown: AIX delivers only when the shopper clicks or lands, so the
  click after the notice is their consent. Decide
  who those shoppers are exactly as you set `country`, including the locale's
  region when there is no country, because AIX applies the rule that way. Store
  links need no special labels. If you already have consent another way, for
  example a consent banner that covers sharing with stores, send
  `"consent": "granted"` in the intent's `privacy` instead. Without consent the
  shopper still goes to the store; AIX just does not share (`consent_required`).
- Sensitive topics are never shared: health and medical (medications,
  conditions, pregnancy and fertility, sexual health, mental health), sexual
  wellness, sexual orientation or gender identity, religion, politics, racial or
  ethnic origin, union membership, immigration status and criminal history.
  AIX answers `suppressed` with `reason: "sensitive_category"` and withdraws
  the session. Carry on normally and never reword a task to get it through.
- When a shopper asks you to delete their data, call the deletion endpoint
  (section 4) for each of their session ids.

## 1. Find the integration points

Locate the server-side code that handles a shopper message, and the code that
produces product or store links.

- Node/TypeScript (Next.js, Express, serverless): add the module below and call
  it from the server handler.
- Python (FastAPI, Django, Flask): make the same two requests with httpx or requests.
- Mobile or browser-only apps: add the calls to the backend the app already
  uses; if there is none, add one small server function. The key stays server-side.
- Assistants that already speak MCP can use the MCP tools below instead.

## 2. Send intent

~~~http
POST https://aix.brambles.ai/api/aix/intents
Authorization: Bearer $AIX_SENDER_KEY
Content-Type: application/json

{
  "session_id": "conv_123",
  "task": {
    "category": "Running shoes",
    "summary": "Wide running shoes for daily training, cushioning preferred, up to $150.",
    "budget": { "currency": "USD", "max_cents": 15000 },
    "facts": [
      { "attribute": "width", "value": "wide", "strength": "required" },
      { "attribute": "use_case", "value": "running", "strength": "required" },
      { "attribute": "feature", "value": "cushioning", "strength": "preferred" }
    ]
  },
  "page_url": "https://reviews.example/best-running-shoes",
  "locale": "en-US",
  "country": "US",
  "privacy": { "gpc": false }
}
~~~

- `session_id`: your conversation id, 1 to 128 characters from `A-Z a-z 0-9 _ . : -`.
  AIX stores only a keyed hash of it.
- Prefer `task` when your agent already understands the request. Otherwise
  send `"message": "<the shopper's message>"` instead of `task` and AIX
  extracts the task.
- `facts[].attribute` is one of brand, width, size, department, use_case,
  feature, merchant, deal. `strength` is required, preferred or excluded.
- `budget`: omit it to keep the current budget; send `null` when the shopper
  drops it. `turn` (optional integer) orders updates; older turns are ignored.
- `country`: required in production (or `locale` with a region). `privacy.consent`:
  `"granted"` only if you already have consent to share with stores, `"denied"`
  (withdraws the session) or `"unknown"`.
- An accepted intent carries `quality` (`score` 0-100 and the `signals` behind it): a
  specific category, a summary with substance, requirements, preferences and a budget
  score high; junk text, vague categories and the same task sent from most of your
  sessions score low. Stores pay only for intents above their quality bar, so send the
  shopper's real task as they refine it.
- Responses: `202` `{"status":"accepted"}`; `200` `{"status":"suppressed","reason":...}`
  where `reason` is `privacy_signal`, `country_required` or
  `sensitive_category` (with `sensitive_class`); `200` `{"status":"no_shopping_intent"}`
  or `{"status":"duplicate_or_stale"}`; `400` invalid body; `401` bad key;
  `429` or `503` retry later. A suppressed intent stores nothing and withdraws
  what the session sent before. Sandbox responses may carry a `warning`
  (`{"code":"country_required",...}`) for something production would refuse.

## 3. Report the handoff

~~~http
POST https://aix.brambles.ai/api/aix/handoffs
Authorization: Bearer $AIX_SENDER_KEY
Content-Type: application/json

{
  "session_id": "conv_123",
  "destination_url": "https://store.example/products/wide-runner",
  "consent": "granted",
  "product": { "title": "Wide Runner 2", "price_cents": 12999, "currency": "USD", "brand": "Example", "merchant": "Example Store" }
}
~~~

`destination_url` is the store's own page URL, even when you send the shopper
through an affiliate or ad link. `consent`: send `"granted"` when the shopper was
shown the sharing notice under your assistant (opening a store link then shares their
request), `"denied"` when they refused (this withdraws the session), or leave it out.
Shoppers in the EEA, the UK or Switzerland are delivered only with consent here or on
the intent; otherwise `status` is `consent_required` and the shopper goes to the store
unshared.

When a participating store would pay (`"status":"delivers_on_click"`), the response
carries `aix_param` (`"aix=<handoff_id>"`). **Keep your own link** (affiliate, ad or
plain) and append `aix_param` to the store URL inside it, before you wrap it, the way
you would add a UTM tag; `landing_url` is the store URL with it appended, for a plain
link. The intent is delivered, and you earn, when the shopper lands on the store with
it; the store claims it there. If you have no link system of your own, you can open
`handoff_url` instead: an AIX link (`https://aix.brambles.ai/h/...`) that delivers on
the shopper's click and always lands on the store page. Never open or prefetch either
from your server; visits from the server that reported the handoff, bots and link
previews are not counted. Otherwise there is no `aix_param`, `handoff_url` is the
original link, and `status` is `not_delivered` (no store pays for this handoff right
now; AIX does not say why, so a key never learns a store's commercial state),
`consent_required` or `delivery_failed`. Report the handoff when you render the link;
if it fails or times out, show your link unchanged.

Limits: the AIX link delivers only when opened within an hour of the report, and a link
opened again from another network address sends the shopper on without the receipt.
A new sender gets at most 100 paid deliveries a day for its first 30
days, then 1,000 a day (Brambles can raise it). Destinations must be https. In the
sandbox the AIX link shows a test page naming the destination instead of redirecting.
A handoff needs an accepted intent for the same `session_id` first
(`409` `no_active_intent` otherwise). A handoff whose product or link is a
sensitive topic answers `"status":"suppressed"` (`sensitive_category`) and
withdraws the session; keep your link unchanged.

**Agentic checkout (UCP, ACP).** If your assistant buys for the shopper through the
store's checkout API instead of sending them to the page, there is no click. Report
the handoff as usual and pass its `handoff_id` in the checkout; the store redeems it
once the checkout completes an order, and you are credited as for a click:

- UCP (create, update or complete): `"attribution": {"aix_handoff": "<handoff_id>"}`.
- ACP (create and complete only): `"affiliate_attribution": {"provider": "aix.brambles.ai",
  "token": "<handoff_id>", "touchpoint": "first"}`, with `"touchpoint": "last"` when
  you attach it at complete.

The store can redeem a handoff once, within an hour of reporting it. Put nothing else
about the shopper in the attribution.

**Personal Agent Protocol (PAP).** A store that receives through AIX can say so in its PAP
file, `https://<store domain>/.well-known/poppy.json`:
`"extensions": {"brambles.ai/aix": {"version": "1"}}`. When your agent talks to such a
store through PAP instead of sending the shopper to a page, report the handoff as usual
(the store's URL as `destination_url`) and put its id in the `context` of your first
message to the store's agent: `"brambles.ai/aix": {"handoff_id": "<handoff_id>"}`. When
the conversation completes an order, the store redeems it and you are credited as for a
click; a conversation without an order earns nothing. Put nothing else about the shopper
in it.

## 4. Honour deletion requests

~~~http
POST https://aix.brambles.ai/api/aix/privacy/delete
Authorization: Bearer $AIX_SENDER_KEY
Content-Type: application/json

{ "session_id": "conv_123" }
~~~

AIX deletes that session's intents and clears its copy of every receipt
delivered from them, then answers
`{"status":"deleted","environment":"live","intents_deleted":2,"receipts_cleared":1}`.
Repeating the call is safe. Billing records stay; they hold no shopper content.
Map a shopper to all of their session ids yourself: AIX only knows sessions.

## 5. Verify in the sandbox

1. Send one intent and one handoff with the sandbox key (any public store URL
   works). The handoff answers `"status":"test_delivered"` with a
   `receiver_preview` (what a store would receive) and the same `landing_url` and
   `aix_param` a production handoff returns, so you can check your link keeps them.
2. `GET https://aix.brambles.ai/api/aix/activity` with the same key shows the accepted intent and the
   delivered test handoff. The AIX console then shows the agent as connected.
3. Tell the user it is connected and that going live is a console step: create a
   production key and set it as `AIX_SENDER_KEY` in production.

## TypeScript reference

~~~ts
const AIX = "https://aix.brambles.ai/api/aix";

async function aix(path: string, body: unknown): Promise<any> {
  const key = process.env.AIX_SENDER_KEY;
  if (!key) return null;
  try {
    const res = await fetch(`${AIX}/${path}`, {
      method: "POST",
      headers: { authorization: `Bearer ${key}`, "content-type": "application/json" },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(3000),
    });
    return res.ok ? await res.json() : null;
  } catch {
    return null; // never block the shopper
  }
}

export const sendIntent = (intent: Record<string, unknown>) => aix("intents", intent);

// The store URL to put in your own link (append it before any affiliate or ad wrapping): with ?aix=<handoff id>
// when a store would pay, unchanged otherwise. consent: "granted" when the shopper was shown your sharing notice.
export async function storeUrlForLink(sessionId: string, storeUrl: string,
  product?: Record<string, unknown>, consent?: "granted" | "denied") {
  const result = await aix("handoffs", { session_id: sessionId, destination_url: storeUrl, product, consent });
  return typeof result?.landing_url === "string" ? result.landing_url : storeUrl;
}

export const deleteSession = (sessionId: string) => aix("privacy/delete", { session_id: sessionId });
~~~

## Agent frameworks (Google ADK, LangChain)

If the assistant is an agent built with a framework, give it two tools: one that shares
the shopper's task when they state or change it, and one that adds AIX to a store URL
before it is shown (your own affiliate or ad wrapping still applies after). The key and
the HTTP calls stay in your server code; the model only fills in the task. `country`,
consent and privacy signals come from your server, never from the model: for shoppers in
the EEA, the UK or Switzerland who see the notice line (see Rules), pass
`"consent": "granted"` on the handoff (the click that delivers it is their consent), and
pass `privacy` (`gpc` when the request carries `Sec-GPC: 1`, `do_not_sell`,
`minor`) on intents.

~~~python
import os
import httpx

AIX = "https://aix.brambles.ai/api/aix"

async def aix(path: str, body: dict) -> dict:
    key = os.environ.get("AIX_SENDER_KEY")
    if not key:
        return {}
    try:
        async with httpx.AsyncClient(timeout=3) as client:
            r = await client.post(f"{AIX}/{path}", json=body, headers={"authorization": f"Bearer {key}"})
        data = r.json() if r.is_success else {}
        return data if isinstance(data, dict) else {}
    except (httpx.HTTPError, ValueError):  # ValueError: a 2xx body that isn't JSON
        return {}  # never block the shopper

def task(category: str, summary: str, budget_max: float | None, currency: str) -> dict:
    t = {"category": category, "summary": summary}
    if budget_max:
        t["budget"] = {"currency": currency, "max_cents": round(budget_max * 100)}
    return t
~~~

Google ADK (`google-adk`): the session id is `tool_context.session.id`; put `country`,
`consent` and `privacy` in the session state when you create the session.

~~~python
from google.adk.agents import Agent
from google.adk.tools import ToolContext

async def share_shopping_task(category: str, summary: str, tool_context: ToolContext,
                              budget_max: float | None = None, currency: str = "USD") -> dict:
    """Share the shopper's current shopping task when they state or change what they want
    to buy. Shopping requirements only: never names, contact details or order numbers."""
    result = await aix("intents", {"session_id": tool_context.session.id,
        "task": task(category, summary, budget_max, currency),
        **({"country": tool_context.state["country"]} if tool_context.state.get("country") else {}),
        **({"privacy": tool_context.state["privacy"]} if tool_context.state.get("privacy") else {})})
    return {"status": result.get("status", "unavailable")}

async def store_link(url: str, tool_context: ToolContext, product_title: str | None = None) -> dict:
    """Call with the store's product URL before showing it to the shopper, and show the returned URL
    instead (wrapped in your affiliate link as usual, if you use one)."""
    result = await aix("handoffs", {"session_id": tool_context.session.id, "destination_url": url,
        **({"consent": tool_context.state["consent"]} if tool_context.state.get("consent") else {}),
        **({"product": {"title": product_title}} if product_title else {})})
    return {"url": result.get("landing_url", url), "handoff_id": result.get("handoff_id")}

agent = Agent(name="shopping_assistant", model=MODEL, instruction=INSTRUCTIONS,
              tools=[share_shopping_task, store_link])
~~~

LangChain (`langchain` 1.x): pass the conversation id, country, consent and privacy signals as the run's context.

~~~python
from dataclasses import dataclass
from langchain.agents import create_agent
from langchain.tools import tool, ToolRuntime

@dataclass
class Shopper:
    session_id: str               # your conversation id
    country: str | None = None    # from your host's geolocation header
    consent: str | None = None    # "granted" when shown the notice (EEA, UK, CH), "denied" on opt-out
    privacy: dict | None = None   # e.g. {"gpc": True} when the request carries Sec-GPC: 1

@tool
async def share_shopping_task(category: str, summary: str, runtime: ToolRuntime[Shopper],
                              budget_max: float | None = None, currency: str = "USD") -> str:
    """Share the shopper's current shopping task when they state or change what they want
    to buy. Shopping requirements only: never names, contact details or order numbers."""
    shopper = runtime.context
    result = await aix("intents", {"session_id": shopper.session_id, "task": task(category, summary, budget_max, currency),
        **({"country": shopper.country} if shopper.country else {}), **({"privacy": shopper.privacy} if shopper.privacy else {})})
    return result.get("status", "unavailable")

@tool
async def store_link(url: str, runtime: ToolRuntime[Shopper], product_title: str | None = None) -> str:
    """Call with the store's product URL before showing it to the shopper, and show the returned URL
    instead (wrapped in your affiliate link as usual, if you use one)."""
    result = await aix("handoffs", {"session_id": runtime.context.session_id, "destination_url": url,
        **({"consent": runtime.context.consent} if runtime.context.consent else {}),
        **({"product": {"title": product_title}} if product_title else {})})
    return result.get("landing_url", url)

agent = create_agent(MODEL, tools=[share_shopping_task, store_link], context_schema=Shopper)
# The tools are async, so run the agent with ainvoke (or astream).
await agent.ainvoke({"messages": messages}, context=Shopper(session_id=conversation_id, country=country))
~~~

TypeScript has the same shape: `FunctionTool` with a zod schema from `@google/adk`
(session id `toolContext?.sessionId`), or `tool()` and `createAgent({ contextSchema })`
from `langchain` (context in the tool's `runtime.context`), calling the module above.

## MCP

Hosts with remote MCP support can connect `https://aix.brambles.ai/api/aix/mcp` with the header
`Authorization: Bearer <key>`, set from an environment variable (never paste the
key into chat). A sender key can call `submit_intent` with an `intent`
object shaped like the intent body above, `report_handoff`, `delete_session`
and `get_activity`.
