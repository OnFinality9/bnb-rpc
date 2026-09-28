# Using BNB Smart Chain JSON-RPC

BNB Smart Chain (BSC) deliberately stays close to the Go-Ethereum JSON-RPC API, so most Ethereum tooling works with little more than a different endpoint and chain ID. The interesting parts are the places where BSC adds its own behavior: fast finality, BSC-specific APIs, and provider choices for high-volume log access.

This tutorial uses an OnFinality public endpoint:

```bash
export BSC_RPC=https://bnb.api.onfinality.io/public
```

## Network identity

| Property | BSC Mainnet |
| --- | --- |
| Chain ID | `56` (`0x38`) |
| Native gas token | BNB |
| JSON-RPC model | Geth-compatible |

Check the chain before doing anything else:

```bash
curl -s "$BSC_RPC" \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
```

Expected result:

```json
{"jsonrpc":"2.0","id":1,"result":"0x38"}
```

## BSC is EVM-compatible, but finality is worth treating explicitly

BSC's documentation describes its own fast-finality mechanism. Applications that only want blocks which have reached finality can use the `finalized` block tag with normal Ethereum-style JSON-RPC methods.

For example:

```bash
curl -s "$BSC_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBlockByNumber",
    "params":["finalized",false]
  }'
```

That is often more meaningful than hard-coding an arbitrary "wait N blocks" rule.

## Read BNB balance

```bash
curl -s "$BSC_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getBalance",
    "params":["0xYOUR_ADDRESS","latest"]
  }'
```

The result is a hexadecimal quantity in wei. BNB uses 18 decimals.

## Read a BEP-20 token like an ERC-20

For application developers, BEP-20 token access normally looks like ERC-20 access. Instead of hand-encoding ABI data, use a contract ABI with an EVM library.

```bash
npm install ethers
```

```js
import {Contract, JsonRpcProvider, formatUnits} from 'ethers';

const provider = new JsonRpcProvider('https://bnb.api.onfinality.io/public');

const token = new Contract(
  '0xTOKEN_CONTRACT',
  [
    'function balanceOf(address) view returns (uint256)',
    'function decimals() view returns (uint8)',
    'function symbol() view returns (string)',
  ],
  provider,
);

const holder = '0xHOLDER_ADDRESS';
const [rawBalance, decimals, symbol] = await Promise.all([
  token.balanceOf(holder),
  token.decimals(),
  token.symbol(),
]);

console.log(`${formatUnits(rawBalance, decimals)} ${symbol}`);
```

Under the hood, these reads are ordinary `eth_call` requests.

## Event logs: why provider choice matters on BSC

BNB Chain's documentation notes that `eth_getLogs` is disabled on the official mainnet endpoints it lists and recommends a third-party endpoint for log access. That matters for explorers, bots, accounting systems, and indexers.

With a provider endpoint, the JSON-RPC shape is still standard:

```bash
curl -s "$BSC_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getLogs",
    "params":[{
      "fromBlock":"0xSTART_BLOCK",
      "toBlock":"0xEND_BLOCK",
      "address":"0xCONTRACT_ADDRESS",
      "topics":[]
    }]
  }'
```

For sustained indexing, use bounded ranges and checkpoint progress. Provider limits on block ranges or response size can be stricter than the protocol itself.

## Inspect a transaction lifecycle

First fetch the transaction:

```bash
curl -s "$BSC_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getTransactionByHash",
    "params":["0xTRANSACTION_HASH"]
  }'
```

Then fetch its receipt:

```bash
curl -s "$BSC_RPC" \
  -H 'content-type: application/json' \
  --data '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"eth_getTransactionReceipt",
    "params":["0xTRANSACTION_HASH"]
  }'
```

Use `status` in the receipt to tell whether EVM execution succeeded. Inclusion in a block does not imply a contract call succeeded.

## Sending transactions

A backend should build and sign the transaction locally. The RPC endpoint only needs the final signed payload:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_sendRawTransaction",
  "params": ["0xSIGNED_TRANSACTION"]
}
```

Before signing, common calls include:

- `eth_getTransactionCount` for the account nonce.
- `eth_estimateGas` for gas limit estimation.
- `eth_gasPrice` or EIP-1559-compatible fee methods supported by your tooling.
- `eth_call` to preview a contract read or reproduce a revert.

## A practical finality pattern

For a payment or deposit watcher, separate these states instead of using one boolean:

1. **Seen** — transaction is known by the node.
2. **Included** — receipt has a block number.
3. **Successful** — receipt `status` indicates successful execution.
4. **Finalized** — the containing block is at or behind the chain's finalized head.

That gives downstream systems a more useful state machine than "confirmed/not confirmed".

## References

- [BNB Chain JSON-RPC endpoints and API notes](https://docs.bnbchain.org/bnb-smart-chain/developers/json_rpc/json-rpc-endpoint/)
- [BNB Smart Chain quick guide](https://docs.bnbchain.org/bnb-smart-chain/developers/quick-guide/)
- [OnFinality BNB Smart Chain RPC](https://onfinality.io/en/networks/bnb-smart-chain)
- [OnFinality network directory](https://onfinality.io/en/networks)
