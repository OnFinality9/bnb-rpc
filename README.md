# BNB Smart Chain RPC Getting Started

This guide shows how to connect to BNB Smart Chain (BSC), verify the chain, read balances and blocks, call contracts, and query event logs.

## RPC endpoint

```text
https://bnb.api.onfinality.io/public
```

BSC is EVM-compatible, so Ethereum JSON-RPC tooling generally works after changing the endpoint and chain ID.

## 1. Verify the chain ID

BNB Smart Chain Mainnet uses chain ID `56` (`0x38`).

```bash
curl -s https://bnb.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

Expected result:

```json
{"jsonrpc":"2.0","id":1,"result":"0x38"}
```

## 2. Get the latest block

```bash
curl -s https://bnb.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'
```

## 3. Read a BNB balance

```bash
curl -s https://bnb.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBalance",
    "params":["0xYOUR_ADDRESS","latest"]
  }'
```

BNB uses 18 decimals, and the RPC result is a hex-encoded wei value.

## 4. Read BEP-20 token data

BEP-20 contracts use the normal EVM `eth_call` pattern:

```bash
curl -s https://bnb.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_call",
    "params":[
      {"to":"0xTOKEN_CONTRACT","data":"0xABI_ENCODED_CALLDATA"},
      "latest"
    ]
  }'
```

Use an ABI-aware library for real applications.

## 5. Query contract events

```bash
curl -s https://bnb.api.onfinality.io/public \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getLogs",
    "params":[{
      "fromBlock":"0xSTART_BLOCK",
      "toBlock":"latest",
      "address":"0xCONTRACT_ADDRESS"
    }]
  }'
```

For indexers, use bounded ranges and persist progress.

## 6. JavaScript example

```js
const RPC_URL = 'https://bnb.api.onfinality.io/public';

async function rpc(method, params = []) {
  const res = await fetch(RPC_URL, {
    method: 'POST',
    headers: {'content-type': 'application/json'},
    body: JSON.stringify({jsonrpc: '2.0', id: 1, method, params}),
  });

  const body = await res.json();
  if (body.error) throw new Error(body.error.message);
  return body.result;
}

const chainId = Number.parseInt(await rpc('eth_chainId'), 16);
const block = Number.parseInt(await rpc('eth_blockNumber'), 16);
console.log({chainId, block});
```

## Wallet settings

| Setting | Value |
| --- | --- |
| Network | BNB Smart Chain Mainnet |
| Chain ID | `56` |
| Native token | BNB |
| RPC | `https://bnb.api.onfinality.io/public` |
| Explorer | `https://bscscan.com` |

## Useful methods

| Method | Use |
| --- | --- |
| `eth_getTransactionByHash` | Fetch a transaction |
| `eth_getTransactionReceipt` | Inspect execution and logs |
| `eth_getTransactionCount` | Read account nonce |
| `eth_gasPrice` | Read gas-price suggestion |
| `eth_estimateGas` | Estimate gas |
| `eth_getCode` | Check contract bytecode |
| `eth_sendRawTransaction` | Broadcast a signed transaction |

## Troubleshooting

### Wrong chain ID

If `eth_chainId` does not return `0x38`, stop before sending transactions and verify the endpoint.

### `eth_getLogs` times out

Split large scans into smaller block windows.

### Nonce errors

If several workers send from the same account, coordinate nonce assignment rather than relying on independent `eth_getTransactionCount` calls.

## Resources

- [BNB Smart Chain documentation](https://docs.bnbchain.org/bnb-smart-chain/)
- [OnFinality BNB Smart Chain RPC](https://onfinality.io/en/networks/bnb)
- [OnFinality RPC network directory](https://onfinality.io/en/networks)
