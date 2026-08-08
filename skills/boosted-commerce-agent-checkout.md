---
name: boosted-commerce-agent-checkout
description: >-
  Take a buyer from cart to completed order on a Boosted Commerce brand storefront through its
  UCP/MCP endpoint, with the buyer approving payment. Covers cart lifecycle, checkout lifecycle,
  idempotent completion, and order retrieval.
generated: '2026-08-08'
method: generated
source: mcp/boosted-commerce-ucp-tools-list.json (live tools/list, 2026-08-08) + https://primelabs.org/agents.md
api: Boosted Commerce Storefront Agent API (UCP / MCP)
transport: JSON-RPC 2.0 over HTTP POST
endpoints:
  - https://primelabs.org/api/ucp/mcp
  - https://www.myvitalvitamins.com/api/ucp/mcp
  - https://happyhealthyhippieco.com/api/ucp/mcp
  - https://www.asterwood.co/api/ucp/mcp
operations:
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Buy from a Boosted Commerce brand as an agent

## The one rule that outranks everything else

**You may not complete payment without explicit, contemporaneous buyer approval.** The provider
states this in its own agent instructions. If you cannot put the buyer in the loop at the moment of
payment, stop at `create_checkout`, hand the buyer the `continue_url` the server returns, or install
`https://shop.app/SKILL.md` and route the purchase through Shop Pay instead.

## Preconditions

- `meta.ucp-agent.profile` on every call, resolvable over HTTPS. Discovery runs before dispatch.
- `context.address_country` and `context.currency` set for correct pricing, tax and availability.
- Confirm the protocol version at `GET <host>/.well-known/ucp` first.

## Steps

1. **Build a cart.** `create_cart` with the line items you resolved via
   `boosted-commerce-shop-catalog`. Keep the returned cart id.
2. **Adjust.** `update_cart` for quantity or line changes; `get_cart` to re-read totals before you
   show anything to the buyer. `cancel_cart` if the buyer walks away.
3. **Open a checkout.** `create_checkout`. The response carries line items, totals, and any
   applicable discounts or taxes.
4. **Fulfill.** `update_checkout` to set the shipping address and shipping method. Note the store's
   fulfillment capability config: single shipping destination, no multi-destination splits.
5. **Confirm with the buyer.** Present the final totals from `get_checkout`. Get an explicit yes.
6. **Complete — idempotently.** `complete_checkout` requires `meta.idempotency-key` alongside
   `meta.ucp-agent`. Generate the key **once per buyer intent** and reuse the same value on every
   retry of that same purchase. Never mint a fresh key on a retry; that is how you double-charge.
   The response returns the order id and the Thank You Page URL.
7. **Follow up.** `get_order` for status. `cancel_checkout` if the buyer backs out before completion.

## Errors

Errors are JSON-RPC 2.0 error objects over HTTP 422, not `application/problem+json`. Branch on
`error.data.code`, not on the numeric `error.code`. Every observed error carries
`error.data.continue_url` — a human-checkout handoff. Use it rather than failing the buyer outright.

| `error.data.code` | Meaning | What to do |
|---|---|---|
| `invalid_profile_url` | No `meta.ucp-agent.profile` was sent | Send a resolvable profile URI |
| `profile_unreachable` | Profile URI sent but not fetchable | Host the profile at a public HTTPS URL |
| HTTP 429 | Per-IP rate limit | Back off and retry with the same idempotency key |

See `errors/boosted-commerce-problem-types.yml`.

## Payment

Payment instruments are supplied through the UCP payment handlers the store declares in
`/.well-known/ucp` — `com.google.pay` and `dev.shopify.card`. You never handle raw card data, and
Boosted Commerce never issues you a credential.
