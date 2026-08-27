---
name: carmd-agentic-purchase
description: >-
  Buy a CarMD product (for example the CarMD Connect vehicle-health device) on behalf of a consenting
  human buyer, using CarMD's live Universal Commerce Protocol MCP endpoint. Covers catalog search, cart
  construction, checkout, the human-approval gate on payment, and how to back out at each stage.
api: CarMD Universal Commerce (UCP) MCP Server
endpoint: https://carmd.com/api/ucp/mcp
operations:
  - search_catalog
  - get_product
  - lookup_catalog
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-27'
method: generated
source: >-
  Grounded in the verbatim tools/list response captured from https://carmd.com/api/ucp/mcp on 2026-08-27
  (mcp/carmd-ucp-tools-list.json). Every tool name above appears in that response; none are invented.
---

# Buying from CarMD as an agent

CarMD serves a live, anonymous MCP endpoint at `https://carmd.com/api/ucp/mcp`. It speaks JSON-RPC 2.0,
reports `protocolVersion: 2024-11-05` and `serverInfo.name: universal-commerce`, and exposes 13 tools.
This skill is for the storefront — CarMD Connect hardware and accessories. It is **not** the CarMD Vehicle
API; that surface (`api.carmd.com`) refused connections when this repo was last probed.

## Before anything else

Every tool call must carry an agent profile:

```json
{"meta": {"ucp-agent": {"profile": "<your resolvable UCP agent profile URI>"}}}
```

Omit it and the server answers HTTP 422 with JSON-RPC error `-32001` / `invalid_profile_url`. This is
identity discovery, not a credential — there is no API key and no OAuth token for the anonymous commerce
surface.

Prices come back as integers in ISO 4217 **minor** units paired with a currency code:
`{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 before quoting anything to a human for
two-decimal currencies; zero-decimal currencies such as JPY are already whole units.

## Steps

1. **Confirm the surface.** `GET https://carmd.com/.well-known/ucp` returns the merchant profile — the
   supported UCP versions (`2026-04-08` current, `2026-01-23` still served), the capability URNs, and the
   payment handlers. Do this once per session, not per call.

2. **Find the product.** Call `search_catalog` with the buyer's intent. If you already have an identifier,
   use `get_product` for one item or `lookup_catalog` for several. Pass buyer context
   (`context.address_country`, `context.currency`) so pricing and availability are correct for the buyer,
   not for your egress IP.

3. **Build the cart.** `create_cart`, then `update_cart` to adjust quantities or lines. `get_cart` reads
   it back. Read the cart back before you show a total to the human — do not compute it yourself.

4. **Open the checkout.** `create_checkout` returns line items, totals, discounts and taxes. Use
   `update_checkout` to set the shipping address and delivery method, then `get_checkout` to confirm the
   final total after tax and shipping.

5. **Get explicit human approval, then complete.** `complete_checkout` takes the money. CarMD's own
   `llms.txt` states that agents must not complete payment without explicit buyer consent obtained at the
   moment of payment. If you cannot get contemporaneous approval, stop here and hand the checkout to the
   human, or route the purchase through the Shop skill (`https://shop.app/SKILL.md`) as CarMD's llms.txt
   recommends.

6. **Confirm.** `get_order` returns the resulting order.

## Backing out

| Stage | Reversal | Window |
|---|---|---|
| Cart built | `cancel_cart` | Any time before checkout submission |
| Checkout open, not paid | `cancel_checkout` | Any time before `complete_checkout` |
| Paid order | No API operation — human return/refund | **30 days from the order delivery date** (CarMD refund policy) |

Once `complete_checkout` succeeds there is no programmatic reversal. The published refund policy
(https://carmd.com/policies/refund-policy) accepts returns within 30 days of delivery, product in new or
unused condition with all original inserts and accessories, excluding shipping and handling; refunds take
up to 10 business days to appear after inspection. Say this to the buyer *before* you call
`complete_checkout`, not after.

## Errors and retries

- `-32001` / `invalid_profile_url` (HTTP 422) — you omitted `meta.ucp-agent.profile`. Add it and retry.
- `429` — the endpoint is rate-limited per IP. Back off; CarMD's llms.txt says so explicitly.
- There is **no idempotency key** on this surface. Do not blind-retry `complete_checkout`. If it times out,
  call `get_checkout` (or `get_order`) and read the actual state before deciding anything.

## Related artifacts

- `mcp/carmd-mcp.yml` — the server manifest and deployment mode
- `mcp/carmd-ucp-tools-list.json` — the verbatim tool set with input schemas
- `mcp/carmd-tool-crosswalk.yml` — how these tools map onto the Storefront GraphQL surface
- `conventions/carmd-conventions.yml` — money format, pagination, reversibility, error envelopes
