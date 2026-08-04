---
name: Read GUNZ chain state
description: Confirm you are on the GUNZ L1 chain and read balances, blocks and transactions from
  the public Gunzilla node without any credential.
api: GUNZ Chain JSON-RPC API
endpoint: https://rpc.gunzchain.io/ext/bc/2M47TxWHGnhNtq6pM5zPXdATBtuqubxn5EPFgFmEawCQr9WFML/rpc
operations:
  - eth_chainId
  - web3_clientVersion
  - eth_blockNumber
  - eth_getBalance
  - eth_getTransactionByHash
  - eth_getLogs
generated: '2026-08-04'
method: generated
source: https://gunbygunz.com/documentation/ + live probes 2026-08-04
---

# Read GUNZ chain state

GUNZ is an Avalanche L1 (subnet) running Subnet-EVM. It answers the standard Ethereum JSON-RPC
surface. No API key, no token, no signup.

## 1. Bind to the right chain first

Always call `eth_chainId` before anything else and refuse to continue if it is not `0xa99b`.

```
POST https://rpc.gunzchain.io/ext/bc/2M47TxWHGnhNtq6pM5zPXdATBtuqubxn5EPFgFmEawCQr9WFML/rpc
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}
-> {"jsonrpc":"2.0","id":1,"result":"0xa99b"}     // 43419 = GUNZ mainnet
```

`0xc0a9` (49321) is the GUNZ **testnet**, served at
`https://subnets.avax.network/gunzilla/testnet/rpc`. Anything else is not GUNZ — stop.

A mirror of mainnet is available at `https://subnets.avax.network/gunzilla/mainnet/rpc`. Use it as a
failover; it returns the same chain ID.

## 2. Read the head

```
{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}
{"jsonrpc":"2.0","id":1,"method":"web3_clientVersion","params":[]}
```

Blocks land every 1–2 seconds. The documentation states finality is reached almost immediately and
recommends waiting **2–3 blocks** before treating a transaction as irreversible. Do not apply
Ethereum-mainnet confirmation depths here.

## 3. Read balances

`eth_getBalance` returns **GUN**, the native gas coin — not an ERC-20. 18 decimals.

```
{"jsonrpc":"2.0","id":1,"method":"eth_getBalance","params":["0x<address>","latest"]}
```

Addresses are plain EVM hex (`0x` + 40 hex chars). GUNZ supports no other address format and no
memo/tag field — if a workflow asks you for a destination tag, it is not a GUNZ workflow.

## 4. Track deposits

Gunzilla documents the supported pattern explicitly: scan with `eth_getLogs` over a block range and
resolve with `eth_getTransactionByHash`. There are no webhooks. For a push feed of token movements,
use the explorer subscription instead — see the `Stream and query GUNZScan` skill.

## 5. Handle errors

Every JSON-RPC failure comes back as **HTTP 200** with an `error` object. Never branch on HTTP status.

| code | meaning | what to do |
|---|---|---|
| -32601 | method not exposed on the public node | check the method name; `debug_`/`admin_` namespaces are off |
| -32602 | bad params | 0x-prefix hashes and quantities; addresses must be 20 bytes |
| -32000 | transaction rejected | insufficient GUN balance, nonce too low, or gas price under the network minimum |

Full catalog: `errors/gunzilla-games-error-codes.yml`.

## 6. Quota

The RPC node returns **no** `x-ratelimit-*` headers, but limits exist. Gunzilla will allow-list
partner service IPs on request to remove them (`https://gunbygunz.com/develop/`). Back off on any
non-200 and keep call volume modest until allow-listed.

## Do not

- Do not attempt to deploy a contract. GUNZ is permissioned; deployment requires Gunzilla to approve
  your address in advance and a written description of the code's purpose.
- Do not hold a private key on behalf of a user to sign GUNZ transactions. See the write skill.
