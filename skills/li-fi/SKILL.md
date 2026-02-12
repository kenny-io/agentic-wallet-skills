---
name: li-fi
description: Cross-chain and same-chain token swaps, bridging, and DeFi operations via LI.FI. Use when you or the user want to bridge, swap across chains, move tokens between networks, find cross-chain routes, or execute multi-chain DeFi workflows. Covers phrases like "bridge ETH to Arbitrum", "swap USDC on Ethereum to ETH on Optimism", "move tokens cross-chain", "find best route across chains".
user-invocable: true
disable-model-invocation: false
allowed-tools: ["Bash(curl *li.quest*)", "Bash(npx awal@latest status*)", "Bash(npx awal@latest balance*)", "Bash(npm install @lifi/sdk*)", "Bash(yarn add @lifi/sdk*)", "Bash(pnpm add @lifi/sdk*)", "Bash(bun add @lifi/sdk*)"]
---

# LI.FI API & SDK Integration

LI.FI provides cross-chain swap and bridge functionality across 30+ blockchain networks via a REST API and a TypeScript/JavaScript SDK (`@lifi/sdk`). The SDK uses the same REST API under the hood. Use the REST API for backend services or non-JS languages; use the SDK for JS/TS projects with built-in execution lifecycle management.

## Base URL

```
https://li.quest/v1
```

## Authentication

API key is **optional** but recommended for higher rate limits.

```bash
# Without API key (lower rate limits)
curl "https://li.quest/v1/chains"

# With API key (higher rate limits)
curl "https://li.quest/v1/chains" \
  -H "x-lifi-api-key: YOUR_API_KEY"
```

**Important**: Never expose your API key in client-side code. Use it only in server-side applications.

## Rate Limits

| Tier | Limit |
|------|-------|
| Without API key | Per IP address |
| With API key | Per API key |

Request an API key from LI.FI for production use with higher limits.

## SDK Installation (TypeScript/JavaScript)

For JS/TS projects, the SDK wraps the REST API with execution lifecycle management, automatic token approvals, chain switching, and exchange rate handling.

```bash
npm install @lifi/sdk
# or: yarn add @lifi/sdk | pnpm add @lifi/sdk | bun add @lifi/sdk
```

```typescript
import { createConfig } from '@lifi/sdk';

createConfig({
  integrator: 'YourAppName', // Required: identifies your application
  apiKey: 'your-api-key',    // Optional: for higher rate limits
  rpcUrls: {                 // Optional: custom RPC endpoints
    1: ['https://your-ethereum-rpc.com'],
    137: ['https://your-polygon-rpc.com'],
  },
  chains: {
    allow: [1, 10, 137, 42161], // Optional: restrict to specific chains
  },
});
```

### SDK Provider Setup

```typescript
// EVM Provider (using viem)
import { createConfig, EVM } from '@lifi/sdk';
import { createWalletClient, http } from 'viem';
import { mainnet } from 'viem/chains';

const walletClient = createWalletClient({ chain: mainnet, transport: http() });

createConfig({
  integrator: 'YourApp',
  providers: [
    EVM({ getWalletClient: () => Promise.resolve(walletClient) }),
  ],
});

// Solana Provider
import { createConfig, Solana } from '@lifi/sdk';

createConfig({
  integrator: 'YourApp',
  providers: [
    Solana({ getWalletAdapter: () => Promise.resolve(walletAdapter) }),
  ],
});
```

## Quick Start (REST API)

### 1. Get a Quote

```bash
curl "https://li.quest/v1/quote?\
fromChain=42161&\
toChain=10&\
fromToken=0xaf88d065e77c8cC2239327C5EDb3A432268e5831&\
toToken=0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1&\
fromAmount=10000000&\
fromAddress=0xYourAddress&\
slippage=0.005"
```

### 2. Approve Token Spending (ERC-20 tokens only)

Before executing a swap with ERC-20 tokens, you must approve the LI.FI contract to spend the token. The quote response includes `estimate.approvalAddress` — this is the contract you need to approve.

```javascript
// Check current allowance
const allowance = await tokenContract.allowance(userAddress, approvalAddress);

// If allowance is insufficient, send an approval transaction
if (BigInt(allowance) < BigInt(fromAmount)) {
  await tokenContract.approve(approvalAddress, fromAmount);
}
```

**Note:** Native tokens (ETH, MATIC, etc.) do not require approval. The SDK handles approvals automatically.

### 3. Execute the Transaction

The quote response includes a `transactionRequest` object with `to`, `data`, `value`, `gasLimit`, `gasPrice`, and `from` fields. Send this transaction using your wallet or web3 library:

```javascript
// Using ethers.js
const tx = await signer.sendTransaction({
  to: quote.transactionRequest.to,
  data: quote.transactionRequest.data,
  value: quote.transactionRequest.value,
  gasLimit: quote.transactionRequest.gasLimit,
});
const receipt = await tx.wait();
```

### 4. Track Status

```bash
curl "https://li.quest/v1/status?\
txHash=0xYourTxHash&\
fromChain=42161&\
toChain=10&\
bridge=stargate"
```

Poll every 5-10 seconds until `status` is `DONE` or `FAILED`.

## Quick Start (SDK)

```typescript
import { getQuote, convertQuoteToRoute, executeRoute } from '@lifi/sdk';

// 1. Get a quote
const quote = await getQuote({
  fromChain: 42161,      // Arbitrum
  toChain: 10,           // Optimism
  fromToken: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831', // USDC on Arbitrum
  toToken: '0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1',   // DAI on Optimism
  fromAmount: '10000000', // 10 USDC (6 decimals)
  fromAddress: '0xYourWalletAddress',
});

// 2. Convert quote to route and execute (approvals handled automatically)
const route = convertQuoteToRoute(quote);
const executedRoute = await executeRoute(route, {
  updateRouteHook(updatedRoute) {
    console.log('Route updated:', updatedRoute);
  },
});
```

## Core Endpoints

### GET /quote

Get a single-step quote with transaction data ready for execution.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fromChain` | number | Yes | Source chain ID |
| `toChain` | number | Yes | Destination chain ID |
| `fromToken` | string | Yes | Source token address |
| `toToken` | string | Yes | Destination token address |
| `fromAmount` | string | Yes | Amount in smallest unit |
| `fromAddress` | string | Yes | Sender wallet address |
| `toAddress` | string | No | Recipient address |
| `slippage` | number | No | Slippage tolerance (0.005 = 0.5%) |
| `integrator` | string | No | Your integrator ID |
| `allowBridges` | string[] | No | Allowed bridges (or `all`, `none`, `default`) |
| `denyBridges` | string[] | No | Denied bridges |
| `preferBridges` | string[] | No | Preferred bridges |
| `allowExchanges` | string[] | No | Allowed exchanges (or `all`, `none`, `default`) |
| `denyExchanges` | string[] | No | Denied exchanges |
| `preferExchanges` | string[] | No | Preferred exchanges |
| `allowDestinationCall` | boolean | No | Allow contract calls on destination (default: true) |
| `order` | string | No | Route preference: `FASTEST` or `CHEAPEST` |
| `fee` | number | No | Integrator fee (0.02 = 2%, max <100%) |
| `referrer` | string | No | Referrer tracking information |
| `fromAmountForGas` | string | No | Amount to convert to gas on destination |
| `maxPriceImpact` | number | No | Max price impact threshold (0.1 = 10%, default: 10%) |
| `skipSimulation` | boolean | No | Skip TX simulation for faster response |
| `swapStepTimingStrategies` | string[] | No | Timing strategy for swap rates |
| `routeTimingStrategies` | string[] | No | Timing strategy for route selection |

**Example Request:**

```bash
curl "https://li.quest/v1/quote?\
fromChain=1&\
toChain=137&\
fromToken=0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48&\
toToken=0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174&\
fromAmount=1000000000&\
fromAddress=0x552008c0f6870c2f77e5cC1d2eb9bdff03e30Ea0&\
slippage=0.005"
```

The response includes `transactionRequest` with all data needed to execute the swap. 
### POST /advanced/routes

Get multiple route options for comparison. Returns routes without transaction data (use `/advanced/stepTransaction` to get TX data for a specific step).

**Request Body:**

```json
{
  "fromChainId": 42161,
  "toChainId": 10,
  "fromTokenAddress": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
  "toTokenAddress": "0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1",
  "fromAmount": "10000000",
  "fromAddress": "0xYourAddress",
  "options": {
    "integrator": "YourAppName",
    "slippage": 0.005,
    "order": "CHEAPEST",
    "bridges": {
      "allow": ["stargate", "hop", "across"]
    },
    "exchanges": {
      "allow": ["1inch", "uniswap"]
    },
    "allowSwitchChain": true,
    "maxPriceImpact": 0.1
  }
}
```

**Example Request:**

```bash
curl -X POST "https://li.quest/v1/advanced/routes" \
  -H "Content-Type: application/json" \
  -d '{
    "fromChainId": 42161,
    "toChainId": 10,
    "fromTokenAddress": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "toTokenAddress": "0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1",
    "fromAmount": "10000000",
    "options": {
      "slippage": 0.005,
      "integrator": "YourApp"
    }
  }'
```

Returns array of routes sorted by `order` preference. Each route contains `steps` but no `transactionRequest` - use `/advanced/stepTransaction` to get TX data.

### POST /advanced/stepTransaction

Populate a step (from `/advanced/routes`) with transaction data. Use this after selecting a route to get the actual transaction to execute.

**Request Body:** Send the full `Step` object from a route.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `skipSimulation` | boolean | No | Skip TX simulation for faster response |

**Example:**

```bash
curl -X POST "https://li.quest/v1/advanced/stepTransaction" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "step-id",
    "type": "cross",
    "tool": "stargate",
    "action": {...},
    "estimate": {...}
  }'
```

**Response:** Returns the Step object with `transactionRequest` populated.

### GET /status

Track transaction status across chains. Can query by sending TX hash, receiving TX hash, or transactionId.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `txHash` | string | Yes | Transaction hash (sending, receiving, or transactionId) |
| `fromChain` | number | No | Source chain ID (speeds up request) |
| `toChain` | number | No | Destination chain ID |
| `bridge` | string | No | Bridge tool name |

For same-chain swaps, set `fromChain` and `toChain` to the same value.

**Example Request:**

```bash
curl "https://li.quest/v1/status?txHash=0x123abc..."
```

**Response includes:** `transactionId`, `sending` (txHash, amount, chainId), `receiving` (txHash, amount, chainId), `status`, `substatus`, `lifiExplorerLink`.

**Status Values:** `NOT_FOUND`, `INVALID`, `PENDING`, `DONE`, `FAILED`

Poll until status is `DONE` or `FAILED`. 

### GET /chains

Get all supported chains.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chainTypes` | string | No | Filter by chain types: `EVM`, `SVM` |

```bash
curl "https://li.quest/v1/chains"
curl "https://li.quest/v1/chains?chainTypes=EVM,SVM"
```

### GET /tokens

Get tokens for specified chains.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chains` | string | No | Comma-separated chain IDs or keys (e.g., `POL,DAI`) |
| `chainTypes` | string | No | Filter by chain types: `EVM`, `SVM` |
| `minPriceUSD` | number | No | Min token price filter (default: 0.0001 USD) |

```bash
curl "https://li.quest/v1/tokens?chains=1,137,42161"
```

### GET /token

Get information about a specific token.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chain` | string | Yes | Chain ID or key (e.g., `POL` or `137`) |
| `token` | string | Yes | Token address or symbol (e.g., `DAI`) |

```bash
curl "https://li.quest/v1/token?chain=POL&token=DAI"
```

### GET /tools

Get available bridges and exchanges. Use this to discover valid tool keys for filtering quotes.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chains` | string | No | Filter by chain IDs |

```bash
curl "https://li.quest/v1/tools"
curl "https://li.quest/v1/tools?chains=1,137"
```

Returns `bridges` and `exchanges` arrays with `key`, `name`, and `supportedChains`. Use these keys in `allowBridges`, `denyBridges`, `allowExchanges`, etc.

**Special values:** `all`, `none`, `default`, `[]`

### GET /connections

Get available token pair connections. At least one filter (chain, token, bridge, or exchange) is required.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fromChain` | string | No | Source chain ID or key |
| `toChain` | string | No | Destination chain ID or key |
| `fromToken` | string | No | Source token address or symbol |
| `toToken` | string | No | Destination token address or symbol |
| `chainTypes` | string | No | Filter by chain types: `EVM,SVM` |
| `allowBridges` | string[] | No | Allowed bridges |
| `denyBridges` | string[] | No | Denied bridges |
| `preferBridges` | string[] | No | Preferred bridges |
| `allowExchanges` | string[] | No | Allowed exchanges |
| `denyExchanges` | string[] | No | Denied exchanges |
| `preferExchanges` | string[] | No | Preferred exchanges |
| `allowSwitchChain` | boolean | No | Include routes requiring chain switch (default: true) |
| `allowDestinationCall` | boolean | No | Include routes with destination calls (default: true) |

```bash
curl "https://li.quest/v1/connections?\
fromChain=POL&\
fromToken=DAI"
```

### POST /quote/contractCalls

Get a quote for bridging tokens and performing multiple contract calls on the destination chain. This enables 1-click DeFi workflows like bridge + swap + deposit.

**Request Body:**

```json
{
  "fromChain": 10,
  "fromToken": "0x4200000000000000000000000000000000000042",
  "fromAddress": "0x552008c0f6870c2f77e5cC1d2eb9bdff03e30Ea0",
  "toChain": 1,
  "toToken": "ETH",
  "toAmount": "100000000000001",
  "integrator": "YourAppName",
  "contractCalls": [
    {
      "fromAmount": "100000000000001",
      "fromTokenAddress": "0x0000000000000000000000000000000000000000",
      "toTokenAddress": "0xae7ab96520de3a18e5e111b5eaab095312d7fe84",
      "toContractAddress": "0xae7ab96520de3a18e5e111b5eaab095312d7fe84",
      "toContractCallData": "0x",
      "toContractGasLimit": "110000"
    },
    {
      "fromAmount": "100000000000000",
      "fromTokenAddress": "0xae7ab96520de3a18e5e111b5eaab095312d7fe84",
      "toTokenAddress": "0x7f39c581f595b53c5cb19bd0b3f8da6c935e2ca0",
      "toContractAddress": "0x7f39c581f595b53c5cb19bd0b3f8da6c935e2ca0",
      "toFallbackAddress": "0x552008c0f6870c2f77e5cC1d2eb9bdff03e30Ea0",
      "toContractCallData": "0xea598cb000000000000000000000000000000000000000000000000000005af3107a4000",
      "toContractGasLimit": "100000"
    }
  ]
}
```

## End-to-End Execution Flow

### Routes vs Quotes

- **`GET /quote`**: Returns single best option with `transactionRequest` ready for immediate execution. Best for simple transfers.
- **`POST /advanced/routes`**: Returns multiple route options for comparison (sorted by `CHEAPEST` or `FASTEST`). Does NOT include `transactionRequest` — use `POST /advanced/stepTransaction` on the chosen step to get TX data.

### Multi-Step Route Execution

Some routes contain multiple steps (e.g., swap on source chain → bridge → swap on destination chain). Each step must be executed sequentially:

1. Call `POST /advanced/routes` to get route options
2. Select a route from the response
3. For each step in the route:
   a. Call `POST /advanced/stepTransaction` with the step to get `transactionRequest`
   b. Approve token spending if needed (check `estimate.approvalAddress`)
   c. Send the transaction
   d. Poll `GET /status` until `DONE`
   e. Move to the next step
4. Some steps may require switching chains (if `allowSwitchChain: true` was set)

### Two-Step Routes and Chain Switching

When `allowSwitchChain` is enabled in route options, the API may return routes that require transactions on two different chains (e.g., swap on Ethereum, then claim on Arbitrum). Your application must:

1. Execute the first step on the source chain
2. Wait for it to complete (poll `/status`)
3. Switch the wallet to the destination chain
4. Execute the second step

### Handling Exchange Rate Changes

Between getting a quote and executing the transaction, exchange rates may change. Best practices:

- **REST API:** Re-fetch the quote (`GET /quote`) immediately before execution. If the new `estimate.toAmount` differs significantly from the original, prompt the user to accept or reject.
- **SDK:** Implement `acceptExchangeRateUpdateHook` to handle this automatically:

```typescript
const executedRoute = await executeRoute(route, {
  acceptExchangeRateUpdateHook(toToken, oldAmount, newAmount) {
    const percentChange = ((parseFloat(newAmount) - parseFloat(oldAmount)) / parseFloat(oldAmount)) * 100;
    // Accept if change is less than 2%, otherwise prompt user
    return Promise.resolve(Math.abs(percentChange) < 2);
  },
  switchChainHook(chainId) {
    // Handle wallet chain switching for 2-step routes
    return walletClient.switchChain({ id: chainId });
  },
  updateRouteHook(updatedRoute) {
    // Persist route state for recovery after crashes/refreshes
    saveRouteState(updatedRoute);
  },
});
```

### Destination Gas (`fromAmountForGas`)

When bridging to a chain where the user has no native gas token, use the `fromAmountForGas` parameter to convert a portion of the bridged amount into native gas on the destination chain:

```bash
curl "https://li.quest/v1/quote?\
fromChain=1&toChain=42161&\
fromToken=0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48&\
toToken=0xaf88d065e77c8cC2239327C5EDb3A432268e5831&\
fromAmount=10000000&\
fromAddress=0xYourAddress&\
fromAmountForGas=1000000"
```

This ensures the user receives some ETH on Arbitrum to pay for future transactions.

## Execution Lifecycle (SDK)

The SDK provides built-in lifecycle management that the REST API requires you to implement manually.

### Resume Interrupted Execution

```typescript
import { resumeRoute } from '@lifi/sdk';

// Resume from the last saved route state (e.g., after page refresh)
const resumedRoute = await resumeRoute(savedRoute, {
  updateRouteHook(route) { saveRouteState(route); },
});
```

### Background Execution

```typescript
import { executeRoute, updateRouteExecution, resumeRoute } from '@lifi/sdk';

// Start execution
const routePromise = executeRoute(route, {
  updateRouteHook(route) { saveRouteState(route); },
});

// Move to background (user navigates away)
updateRouteExecution(route, { executeInBackground: true });

// Later: resume in foreground
const resumed = await resumeRoute(savedRoute, { executeInBackground: false });
```

### Stop and Track Active Routes

```typescript
import { stopRouteExecution, getActiveRoutes, getActiveRoute } from '@lifi/sdk';

// Stop execution (already-sent transactions still complete on-chain)
const stoppedRoute = stopRouteExecution(route);

// Get all active routes
const activeRoutes = getActiveRoutes();

// Get a specific active route
const activeRoute = getActiveRoute(routeId);
```

## Integration Examples

### Python

```python
import requests

BASE_URL = "https://li.quest/v1"

def get_quote(from_chain, to_chain, from_token, to_token, amount, address):
    response = requests.get(f"{BASE_URL}/quote", params={
        "fromChain": from_chain,
        "toChain": to_chain,
        "fromToken": from_token,
        "toToken": to_token,
        "fromAmount": amount,
        "fromAddress": address,
        "slippage": 0.005
    })
    return response.json()

def get_status(tx_hash, from_chain, to_chain, bridge):
    response = requests.get(f"{BASE_URL}/status", params={
        "txHash": tx_hash,
        "fromChain": from_chain,
        "toChain": to_chain,
        "bridge": bridge
    })
    return response.json()

# Example usage
quote = get_quote(
    from_chain=42161,
    to_chain=10,
    from_token="0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    to_token="0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1",
    amount="10000000",
    address="0xYourAddress"
)

print(f"Estimated output: {quote['estimate']['toAmount']}")
print(f"Gas cost: {quote['estimate']['gasCosts']}")
```

### Node.js (fetch)

```javascript
const BASE_URL = 'https://li.quest/v1';

async function getQuote(params) {
  const searchParams = new URLSearchParams(params);
  const response = await fetch(`${BASE_URL}/quote?${searchParams}`);
  return response.json();
}

async function getRoutes(body) {
  const response = await fetch(`${BASE_URL}/routes`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  return response.json();
}

async function pollStatus(txHash, fromChain, toChain, bridge) {
  const params = new URLSearchParams({ txHash, fromChain, toChain, bridge });
  
  while (true) {
    const response = await fetch(`${BASE_URL}/status?${params}`);
    const data = await response.json();
    
    if (data.status === 'DONE' || data.status === 'FAILED') {
      return data;
    }
    
    await new Promise(resolve => setTimeout(resolve, 5000));
  }
}

// Example usage
const quote = await getQuote({
  fromChain: 42161,
  toChain: 10,
  fromToken: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831',
  toToken: '0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1',
  fromAmount: '10000000',
  fromAddress: '0xYourAddress',
  slippage: 0.005,
});
```

### TypeScript SDK

```typescript
import { getQuote, getRoutes, getChains, getTokens, getToken, getTools, getConnections, convertQuoteToRoute, executeRoute, ChainType } from '@lifi/sdk';

// Get all supported chains
const allChains = await getChains();
const evmChains = await getChains({ chainTypes: [ChainType.EVM] });

// Get tokens for specific chains
const tokens = await getTokens({ chains: [1, 137] });
const usdc = await getToken(1, '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48');

// Get available bridges and DEXs
const tools = await getTools();
console.log('Bridges:', tools.bridges);
console.log('Exchanges:', tools.exchanges);

// Find connections from ETH on Ethereum
const connections = await getConnections({
  fromChain: 1,
  fromToken: '0x0000000000000000000000000000000000000000',
});

// Get multiple routes for comparison
const routeResult = await getRoutes({
  fromChainId: 42161,
  toChainId: 10,
  fromTokenAddress: '0xaf88d065e77c8cC2239327C5EDb3A432268e5831',
  toTokenAddress: '0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1',
  fromAmount: '10000000',
  fromAddress: '0xYourAddress',
  options: {
    slippage: 0.005,
    order: 'CHEAPEST',
    maxPriceImpact: 0.3,
    bridges: { allow: ['stargate', 'hop'], deny: ['multichain'], prefer: ['stargate'] },
    exchanges: { allow: ['1inch', 'uniswap'] },
    allowSwitchChain: true,
  },
});

// Execute with full lifecycle hooks
const executedRoute = await executeRoute(routeResult.routes[0], {
  updateRouteHook(updatedRoute) {
    const step = updatedRoute.steps[0];
    const process = step?.execution?.process;
    const latest = process?.[process.length - 1];
    console.log('Status:', latest?.status, 'TX:', latest?.txHash);
  },
  acceptExchangeRateUpdateHook(toToken, oldAmount, newAmount) {
    const change = ((parseFloat(newAmount) - parseFloat(oldAmount)) / parseFloat(oldAmount)) * 100;
    return Promise.resolve(Math.abs(change) < 2);
  },
  async switchChainHook(chainId) {
    await walletClient.switchChain({ id: chainId });
    return walletClient;
  },
  async updateTransactionRequestHook(txRequest) {
    // Optionally boost gas by 20%
    return {
      ...txRequest,
      gas: txRequest.gas ? BigInt(txRequest.gas) * 120n / 100n : undefined,
    };
  },
});

// Contract calls on destination chain (bridge + swap + deposit in one TX)
import { getContractCallsQuote } from '@lifi/sdk';

const contractCallQuote = await getContractCallsQuote({
  fromAddress: '0xYourAddress',
  fromChain: 10,
  fromToken: '0x0000000000000000000000000000000000000000',
  toAmount: '8500000000000',
  toChain: 8453,
  toToken: '0x0000000000000000000000000000000000000000',
  contractCalls: [{
    fromAmount: '8500000000000',
    fromTokenAddress: '0x0000000000000000000000000000000000000000',
    toContractAddress: '0xTargetContract',
    toContractCallData: '0x...', // Encoded function call
    toContractGasLimit: '210000',
  }],
});
```

## Error Handling

### REST API Errors

| HTTP Status | Description |
|-------------|-------------|
| 200 | Success |
| 400 | Bad request - check parameters |
| 429 | Rate limit exceeded - wait or use API key |
| 500 | Internal error - retry |

### SDK Error Handling

```typescript
try {
  const result = await executeRoute(route, {
    acceptExchangeRateUpdateHook: () => Promise.resolve(true),
  });
} catch (error) {
  if (error.message.includes('Exchange rate has changed')) {
    // Rate changed and was rejected by the hook
  } else if (error.message.includes('insufficient funds')) {
    // Wallet balance too low
  } else if (error.message.includes('user rejected')) {
    // User cancelled in wallet
  } else {
    console.error('Execution failed:', error);
  }
}
```

## Best Practices

1. **Always specify slippage** - Default 0.5% (0.005) works for most cases. Increase for volatile tokens or low-liquidity pairs.

2. **Poll status endpoint** - For cross-chain transfers, poll every 5-10 seconds until `DONE` or `FAILED`.

3. **Handle rate limits** - Implement exponential backoff for 429 responses.

4. **Cache chain/token data** - `/chains` and `/tokens` responses change infrequently.

5. **Use integrator parameter** - Always include `integrator` for analytics and potential monetization.

6. **Validate addresses** - Ensure token addresses match the specified chain.

7. **Handle gas estimation** - The API provides gas estimates, but actual costs may vary. Use `updateTransactionRequestHook` in the SDK to adjust gas.

8. **Approve tokens before swapping** - ERC-20 tokens require an `approve` transaction to the `estimate.approvalAddress` before execution. The SDK handles this automatically.

9. **Persist route state** - Use `updateRouteHook` to save route state for recovery after page refreshes, app restarts, or crashes. Use `resumeRoute` to continue.

10. **Handle exchange rate changes** - Rates change between quoting and executing. Re-fetch quotes before execution (REST API) or implement `acceptExchangeRateUpdateHook` (SDK).

11. **Handle chain switching** - For cross-chain routes with `allowSwitchChain: true`, implement `switchChainHook` in the SDK or manually switch chains in your REST API integration.