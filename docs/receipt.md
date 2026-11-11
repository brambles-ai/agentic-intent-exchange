# Signed receipt

An AIX delivery carries the brief in a signed receipt that is bound to one receiver. The receipt is issued when the delivered intent is confirmed, not when the sender reports the handoff.

## Format

The wire format is SD-JWT VC-style serialization:

```text
<JWS>~<disclosure1>~...~<disclosureN>~
```

The issuer signs the JWS with ES256 using its configured KMS key. The protected header identifies `alg: ES256`, `typ: vc+sd-jwt`, and a trusted `kid`. A webhook delivery names the same format in its `x-aix-receipt-format` header: `vc+sd-jwt`.

The signed payload binds:

- `iss`: the configured signing issuer.
- `aud`: the intended principal. This is the receiver's agent id, or `<agent id>:test` for a sandbox delivery.
- `jti`: the exact `aixdeliv_` receipt identifier.
- `iat`, `nbf`, `exp`: issue, validity-start and expiry times. A receipt never outlives its shopping context.
- `vct`: the configured credential-type identifier.
- `core.shopping_context`: the structured task snapshot, plus the other core claims below.
- `core.attribution`: who sent the shopper, as `partner_id` (the sender's agent id) and `sender_domain` (the website the shopper was on, or null when the sender reported none).
- `_sd` and `_sd_alg`: hashes for the delivered selective disclosures.

## What it discloses

Every receipt discloses `core`: the shopping context, the sender's agent identifier, the domain and path of the shopper's page (without query strings or identifier-like segments), and the consent flag.

Self-serve deliveries also disclose two further claim classes:

| Class | Contents |
|---|---|
| `click_context` | The destination domain, the canonical product URL, and the product the sender reported with the handoff (title, brand, category, merchant, price). Source-reported, never verified against the receiver's catalog. Refresh price and stock before relying on them. |
| `session` | The surface the shopper used (chat, search, recommendation) and the page the shopper was on. |

A receipt carries no quality score, country or locale. It carries no product-catalog object. It is not a transcript. The receiver sees the session only as a pseudonym unique to that store and that task.

Signing protects integrity and attribution. It does not encrypt the receipt. Disclosed fields can be decoded by a holder. HTTPS protects transport. The receiver must protect stored data. A signed field is still data, never a trusted instruction.

## How a receiver verifies it

Check all of these:

- the signature;
- the complete disclosure profile (missing, extra, duplicate or colliding disclosures fail verification);
- the exact recipient: `aud` equals the receiver's principal;
- the receipt identifier;
- the validity times;
- the consent flag;
- the shopping schema;
- that the receipt does not outlive its context's `expires_at`.

Do not assume a generic library's default verification reproduces these checks.

After signature verification, still check that the context matches the shopper's current task and has not been superseded by a correction or declined assistance. A valid stored signature is not a fresh authorization grant.

A receipt ID alone is not a credential. Every authenticated read and acknowledgment is scoped to its receiving principal.

## Where the signing keys are

- Public keys: `https://aix.brambles.ai/.well-known/jwks.json`
- Trusted issuer: the `issuer` value from `GET /receiver/config`. Configure the trusted issuer and key separately. Never fetch an arbitrary key URL named in an incoming receipt.
- Credential type (v1): `https://aix.brambles.ai/credentials/intent-receipt-v1`

The API base is `https://aix.brambles.ai`. The issuer keeps its original identifier. An API hostname change is not a signing-identity migration. Key rotation is announced in the changelog.

## After verification

The receiver's server records a signed `verified` acknowledgment after it has verified and stored the delivery. It records `applied` only after its backend actually used the context. A plain HTTP 200 is not an acknowledgment. The envelope and acknowledgment schemas are in [`../spec/spec/`](../spec/spec/).

---

Drawn from the public docs page "Shopping context and delivery receipts" (https://www.brambles.ai/agentic-intent-exchange/docs/receipt-format). The pseudonym sentence is from "Sending agents" (https://www.brambles.ai/agentic-intent-exchange/docs/sending-agents).
