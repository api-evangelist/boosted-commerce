---
name: boosted-commerce-shop-catalog
description: >-
  Search and inspect products in a Boosted Commerce brand storefront (Prime Labs, Vital Vitamins,
  Happy Healthy Hippie, Asterwood) through its anonymous UCP/MCP endpoint. Read-only — no cart, no
  payment.
generated: '2026-08-08'
method: generated
source: mcp/boosted-commerce-ucp-tools-list.json (live tools/list, 2026-08-08)
api: Boosted Commerce Storefront Agent API (UCP / MCP)
transport: JSON-RPC 2.0 over HTTP POST
endpoints:
  - https://primelabs.org/api/ucp/mcp
  - https://www.myvitalvitamins.com/api/ucp/mcp
  - https://happyhealthyhippieco.com/api/ucp/mcp
  - https://www.asterwood.co/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Browse a Boosted Commerce brand catalog

All four Boosted Commerce brand storefronts expose the identical 13-tool UCP shopping service. Pick
the endpoint for the brand you want; the tool names and schemas are the same on every one.

## Before you call anything

1. `GET <host>/.well-known/ucp` and confirm `ucp.version` is a version you support (`2026-04-08` or
   `2026-01-23` today).
2. Every tool call must carry `meta.ucp-agent.profile` — a **publicly fetchable HTTPS URI** for your
   agent profile. The server fetches it before it dispatches the method. If it is missing or
   unreachable you get `-32001 / invalid_profile_url` or `-32001 / profile_unreachable` with HTTP
   422, whatever the method was.
3. Pass `context.address_country` and `context.currency` so prices and availability come back right.
4. No credentials are required for anything in this skill.

## Steps

1. **Search.** Call `search_catalog` with a natural-language query, structured filters, or both — at
   least one is required. Results are cursor-paginated and the first page is deliberately small.
2. **Page.** When the buyer asks for more, resend `search_catalog` with the `pagination.cursor`
   returned by the previous response.
3. **Resolve identifiers in bulk.** When you already hold product or variant IDs
   (`gid://shopify/Product/…`, `gid://shopify/ProductVariant/…`), call `lookup_catalog` once for all
   of them rather than looping. Results come back grouped by product; a variant match returns its
   parent product with the exact variant flagged.
4. **Get detail before recommending.** Call `get_product` for the single product you are about to
   present. It is the only tool that returns exact pricing and real-time availability, and it
   supports `selected` / `preferences` for narrowing variants interactively.

## Rules

- Back off on HTTP 429. The endpoint is rate-limited per IP.
- Do not scrape the storefront HTML to work around a failed call — the read-only JSON endpoints
  (`/products/{handle}.json`, `/collections/{handle}/products.json`, `/search?q=…&type=product`) are
  published and unauthenticated.
- This skill never mutates state. Anything involving a cart or payment belongs to
  `boosted-commerce-agent-checkout`.
