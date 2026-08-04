---
name: Submit a GUNZ transaction safely
description: Broadcast a pre-signed transaction to the GUNZ L1 chain with correct nonce handling,
  gas quoting, retry-safe idempotency and confirmation depth.
api: GUNZ Chain JSON-RPC API
endpoint: https://rpc.gunzchain.io/ext/bc/2M47TxWHGnhNtq6pM5zPXdATBtuqubxn5EPFgFmEawCQr9WFML/rpc
operations:
  - eth_chainId
  - eth_getTransactionCount
  - eth_gasPrice
  - eth_estimateGas
  - eth_sendRawTransaction
  - eth_getTransactionByHash
  - eth_getTransactionReceipt
  - eth_blockNumber
generated: '2026-08-04'
method: generated
source: https://gunbygunz.com/documentation/ + live probes 2026-08-04
---

# Submit a GUNZ transaction safely

This is the only write path Gunzilla exposes publicly. It moves value. Treat every step as
consequential.

## 0. Key custody — the hard rule

GUNZ has no API keys and no OAuth. The only authentication for a write is an **ECDSA secp256k1
signature by the sending account's private key**. An agent must never hold, request, or generate that
key. Accept an already-signed raw transaction from the user's wallet, or hand the unsigned payload
back for the user to sign. Broadcasting is the only part of this flow an agent should own.

## 1. Confirm the chain

```
{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}   -> "0xa99b"  (43419)
```

Chain ID 43419 must be baked into the signature (EIP-155). This is what makes a GUNZ transaction
un-replayable on Avalanche C-Chain or any other EVM chain — and what makes testnet (49321) safe to
rehearse on.

## 2. Get the nonce — this is your idempotency key

```
{"jsonrpc":"2.0","id":1,"method":"eth_getTransactionCount","params":["0x<from>","pending"]}
```

GUNZ has no `Idempotency-Key` header. The per-account nonce *is* the idempotency contract:

- Re-broadcasting the **same signed bytes** is a safe no-op — the node rejects it as already known.
- A transaction reusing an **already-consumed** nonce is rejected outright. Gunzilla lists "a lower
  nonce" as an explicit failure condition.

So on a lost response: **re-send the identical raw transaction**, do not re-sign with a new nonce.
Then reconcile with `eth_getTransactionByHash`.

## 3. Quote gas

```
{"jsonrpc":"2.0","id":1,"method":"eth_gasPrice","params":[]}
{"jsonrpc":"2.0","id":1,"method":"eth_estimateGas","params":[{...}]}
```

Fees are paid in **GUN**, the native coin (18 decimals), not an ERC-20. A gas price below the network
minimum is the third documented failure condition. Live gas guidance is also readable from
`GET https://gunzscan.io/api/v2/stats` (`gas_prices.slow|average|fast`).

## 4. Broadcast

```
{"jsonrpc":"2.0","id":1,"method":"eth_sendRawTransaction","params":["0x<signed>"]}
```

HTTP is 200 even on failure. Check for an `error` object:

| symptom | cause | fix |
|---|---|---|
| `-32000` insufficient funds | not enough GUN for value + gas | top up; on testnet ask Gunzilla to fund the address |
| `-32000` nonce too low | nonce already consumed | re-read `eth_getTransactionCount`, do not blind-retry with a new nonce |
| `-32000` underpriced | gas price below minimum | re-quote with `eth_gasPrice` |
| rejected deployment | GUNZ is permissioned | contract deployment needs prior address approval by Gunzilla |

## 5. Confirm

Poll `eth_getTransactionReceipt`. Blocks land in ~1–2 seconds and Gunzilla states finality is reached
almost immediately — **2–3 blocks is sufficient** to treat a transaction as irreversible. Do not wait
for Ethereum-style depth, and do not treat a single block as final either.

Check `status` on the receipt: `0x1` succeeded, `0x0` reverted. A reverted transaction still consumed
gas — report it as a failure, not as a pending state.

## 6. Rehearse on testnet first

Chain 49321 at `https://subnets.avax.network/gunzilla/testnet/rpc`. There is no self-service faucet:
Gunzilla funds addresses on request ("provide us with your wallet addresses, and we will fund them
for you"). See `sandbox/gunzilla-games-sandbox.yml`.

## Do not

- Do not sign on the user's behalf.
- Do not re-sign a stuck transaction with a fresh nonce without explicit user approval — that can
  produce two settled transfers.
- Do not assume ERC-20 semantics for GUN. It is the native coin; there is no token contract to
  `approve`.
