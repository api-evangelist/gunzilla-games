---
name: Query and stream the GUNZScan explorer
description: Use the three GUNZScan surfaces (GraphQL, REST v2, Etherscan-compatible) to look up
  addresses, blocks, transactions and token transfers, and subscribe to token-transfer events.
api: GUNZScan Explorer API
endpoint: https://gunzscan.io
operations:
  - RootQueryType.address
  - RootQueryType.addresses
  - RootQueryType.block
  - RootQueryType.transaction
  - RootQueryType.tokenTransfers
  - RootQueryType.tokenTransferTxs
  - RootSubscriptionType.tokenTransfers
  - GET /api/v2/stats
  - GET /api/v2/blocks
  - GET /api/v2/transactions/{hash}
  - GET /api/v2/addresses/{hash}
generated: '2026-08-04'
method: generated
source: graphql/gunzilla-games-gunzscan.graphql + live probes 2026-08-04
---

# Query and stream the GUNZScan explorer

GUNZScan is the GUNZ block explorer named in Gunzilla's own chain documentation. It runs Blockscout
and exposes **three different APIs on one host**, each with its own envelope, its own error shape and
its own quota. Pick one deliberately.

| surface | base | shape | quota |
|---|---|---|---|
| GraphQL | `https://gunzscan.io/api/v1/graphql` | Relay connections | 500 / window |
| REST v2 | `https://gunzscan.io/api/v2` | `{items, next_page_params}` | 3000 / window |
| Etherscan-compat | `https://gunzscan.io/api?module=&action=` | `{status, message, result}` | — |

All are anonymous and CORS-open (`access-control-allow-origin: *`).

## 1. Prefer GraphQL for entity reads

The schema is in `graphql/gunzilla-games-gunzscan.graphql` (introspected live). Console:
`https://gunzscan.io/graphiql`.

```graphql
query {
  address(hash: "0x0000000000000000000000000000000000000000") {
    hash
    nonce
    fetchedCoinBalance
    transactionsCount
    tokenTransfersCount
    smartContract { name compilerVersion verifiedViaSourcify }
  }
}
```

Paging is Relay-style — `first`/`after` plus `pageInfo { hasNextPage endCursor }`:

```graphql
query {
  tokenTransfers(tokenContractAddressHash: "0x<token>", first: 25) {
    edges { cursor node { transactionHash fromAddressHash toAddressHash amount tokenIds } }
    pageInfo { hasNextPage endCursor }
  }
}
```

**Complexity ceiling: 100.** This is the single trap on this endpoint. The standard one-shot GraphQL
`IntrospectionQuery` is rejected with `"Field __schema is too complex"`. Introspect one type at a
time with `__type(name: "Transaction")`, and keep `first`/`last` small.

## 2. Use REST v2 for lists and stats

```
GET https://gunzscan.io/api/v2/stats
GET https://gunzscan.io/api/v2/blocks
GET https://gunzscan.io/api/v2/transactions/{hash}
GET https://gunzscan.io/api/v2/addresses/{hash}
```

Page by echoing the previous response's `next_page_params` object back as query parameters; stop when
it is `null`. Page size is server-controlled (50).

Numbers here are **decimal strings**. On the JSON-RPC node the same values are **0x hex**. Do not mix
the two without converting.

## 3. Subscribe instead of polling for token transfers

The schema declares exactly one subscription — `tokenTransfers` — delivered over the Phoenix
WebSocket at `wss://gunzscan.io/socket/websocket?vsn=2.0.0` (a plain GET returns `426 Upgrade
Required`, confirming it is live). Payload is a `TokenTransfer`. There is no block or transaction
subscription: poll `/api/v2/blocks` for those.

Details: `asyncapi/gunzilla-games-gunzscan-events.yml`.

## 4. Errors differ per surface — check the right thing

- REST v2 validation → **HTTP 422**, `{"errors":[{"title","source":{"pointer"},"detail"}]}`
- REST v2 miss → **HTTP 404**, `{"message":"Not found"}`
- Etherscan-compat → **HTTP 200 always**; failure is `{"status":"0", ...}`. Branch on `status`, never
  on the HTTP code.
- GraphQL → **HTTP 200**, `{"errors":[{"message","locations"}]}`

## 5. Respect the quota, and note it is not one quota

Read `x-ratelimit-limit` / `x-ratelimit-remaining` / `x-ratelimit-reset` on every response. REST v2
allows 3000, GraphQL allows 500 — six times tighter. Budget them separately. Every response also
carries `x-request-id`; log it, it is the only correlation handle Gunzilla exposes.

## Do not

- Do not treat a 404 as "not on chain" without confirming you queried GUNZ — check
  `/api/v2/config/backend-version` or the chain ID via the RPC node first.
- Do not expect an OpenAPI document. `https://gunzscan.io/api-docs` is a server-rendered Swagger UI
  with no downloadable spec; `/swagger.json` and `/openapi.json` both 404.
