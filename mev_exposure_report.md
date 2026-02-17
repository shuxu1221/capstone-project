# MEV Exposure Report (Payment Service)

## Scope and context

This report assesses MEV exposure for the payment execution path implemented in `PaymentsService`, focusing on the on-chain transfer flow in `executePayment()` and transaction construction in `buildUnsignedTransaction()`. The goal is to determine what MEV vectors are realistically applicable given the current behavior, what the likely impact is, and what practical mitigations follow directly from the code.

For the purposes of this report, “MEV exposure” means the extent to which a transaction can be observed, reordered, censored/delayed, or used by third parties (searchers/builders/validators) to extract value or impose additional cost/latency on the sender.

## What the code actually does on-chain

The on-chain transaction being signed and broadcast is an ERC-20 `transfer(to, amount)` call for USDC or USDT. There is no swap, no DEX interaction, no price-sensitive execution, and no multi-step atomic sequence on-chain. The payment is effectively a single token transfer from a custodial wallet controlled via Turnkey.

The key portion is visible in `buildUnsignedTransaction()`:

```ts
// Get contract address based on currency
let contractAddress: string;
switch (currency.toUpperCase()) {
  case "USDC":
    contractAddress = env.USDC_ADDRESS;
    break;
  case "USDT":
    contractAddress = env.USDT_ADDRESS;
    break;
  default:
    throw new BadRequestException(`Unsupported currency: ${currency}`);
}

const transferData = encodeFunctionData({
  abi: erc20Abi,
  functionName: "transfer",
  args: [checksummedToAddress, amount],
});
```

Then the transaction parameters are constructed using an RPC gas price quote and an `estimateGas` call with a 10% buffer:

```ts
const [nonce, gasPrice] = await Promise.all([
  publicClient.getTransactionCount({ address: checksummedAddress }),
  publicClient.getGasPrice(),
]);

const gasEstimate = await publicClient.estimateGas({
  account: checksummedAddress,
  to: contractAddress as `0x${string}`,
  data: transferData,
});

const gasLimit = (gasEstimate * 110n) / 100n; // Add 10% buffer

const transactionRequest = {
  to: contractAddress as `0x${string}`,
  value: 0n,
  data: transferData,
  nonce,
  gasPrice,
  gas: gasLimit,
  chainId: publicClient.chain.id,
};

return serializeTransaction(transactionRequest);
```

Finally, `executePayment()` signs the unsigned transaction with Turnkey and submits the raw signed transaction via `blockchainService.submitSignedTransaction(...)`:

```ts
const unsignedTransaction = await this.buildUnsignedTransaction(
  sourceAccount.cryptoWalletAddress,
  destinationAccount.cryptoWalletAddress,
  BigInt(transferStep.fromAmount),
  payment.fromCurrency || "USDC",
);

// Sign transaction with Turnkey using the provided stamp
signedTransaction = await this.signTransactionWithTurnkey(
  workspace.turnkeySubOrgId,
  sourceAccount.cryptoWalletAddress,
  unsignedTransaction,
  signature,
  payment.fromNetwork as NetworkKey,
);

// Submit signed transaction to blockchain
const txHash = await this.blockchainService.submitSignedTransaction(
  signedTransaction,
  destinationAccount.cryptoWalletAddress,
  BigInt(transferStep.fromAmount),
  payment.fromCurrency || "USDC",
  payment.fromNetwork as NetworkKey,
);
```

This means MEV analysis should be centered on:

1. visibility and ordering/inclusion dynamics (mempool / builder behavior),
2. fee bidding strategy (risk of overpaying or being delayed),
3. nonce and retry/replace patterns (risk of unintended replacement chains),
   and not on classic DEX sandwich attacks (because there is no price impact mechanism here).

## MEV exposure: what is low risk vs what is realistic

### 1) Value extraction via sandwich/arbitrage is low for this flow

The canonical MEV patterns that extract value from users (sandwiching, backrunning swaps, oracle manipulation around execution) generally require a transaction to move a market price or create an arbitrageable state transition. In this code path, the transaction is a simple ERC-20 transfer. A third party cannot “sandwich” a transfer in a way that changes the sender’s execution price, because there is no price function being executed.

As a result, the risk of direct value extraction from the transaction’s semantics is low.

That said, “low” does not mean “zero.” There are still indirect ways MEV ecosystem behavior can impose costs:

* paying more than necessary to get included,
* being delayed or not included promptly under certain network conditions,
* revealing payment flows that can be used for surveillance or targeted behavior.

These are not sandwich/arbitrage MEV, but they still fall under “MEV exposure” in practice because they arise from the same ordering/inclusion marketplace.

### 2) Exposure to inclusion delay and fee outliers is realistically present

The transaction fee strategy uses:

```ts
publicClient.getGasPrice()
```

and constructs a legacy `gasPrice` transaction rather than using explicit EIP-1559 style fee caps (`maxFeePerGas` and `maxPriorityFeePerGas`). On chains like Polygon, fee conditions can change quickly. A single-point gas price quote can be:

* too high (leading to systematic overpayment),
* too low (leading to long pending times),
* volatile across time-of-day (creating measurable outliers).

From an MEV perspective, this is exposure to the inclusion marketplace: if the bid is not aligned with current builder/validator preferences, inclusion becomes uncertain. Practically, the system will either pay more than needed to avoid delay (cost leakage) or suffer longer landing times (latency exposure). Your team’s dashboard tasks around “how long does it take our transactions to land” and “outliers + comparison to blockchain avgs” fit directly here.

### 3) Mempool visibility (privacy) is a meaningful exposure for payments

Even when the transfer itself is not exploitable via arbitrage, a public broadcast reveals:

* sender address (custodial wallet),
* receiver address,
* token contract,
* amount,
* timing patterns.

This is important because payments are often more sensitive than trading. If a single custodial wallet is used for many customers, on-chain observers can correlate flows across customers. The code currently uses stablecoin transfer contracts and customer-specific destinations, which makes transactions easy to classify as “payment transfers” from a known service wallet.

Nothing in the shown code suggests private submission or bundle-only delivery; it is built as a standard signed transaction broadcast via an RPC. Under that assumption, privacy exposure is real and ongoing.

### 4) Nonce handling can amplify fee/latency problems

Nonce is fetched here:

```ts
publicClient.getTransactionCount({ address: checksummedAddress })
```

Depending on the client defaults, `getTransactionCount` may use `latest` rather than `pending`. If there are pending transactions from the same custodial wallet, using a `latest` nonce can cause collisions or unintended replacement patterns. The immediate code snippet does not show any explicit retry/replace logic, but in real systems, “stuck tx” handling often exists elsewhere (workers, queue retries, etc.). If replacement happens, the system can enter fee escalation loops, which is another common source of fee outliers.

This is less about adversarial MEV and more about operational exposure to the same mempool auction dynamics. It still belongs in an MEV exposure report because it affects how the transaction competes for inclusion.

## Why QuickNode/RPC utilities matter to MEV exposure (as implied by the task list)

Your weekly task list asks: “What exactly is happening when we estimate gas currently” and “Does QuickNode have any utilities we are or can leverage?”

From the code, gas estimation consists of:

* RPC gas price quote (`getGasPrice`)
* RPC execution simulation (`estimateGas`)
* a fixed 10% buffer

This is simple and workable, but it does not explicitly incorporate:

* fee history distribution,
* percentile-based fee selection,
* predictive landing-time targeting,
* tracking mempool conditions.

RPC providers (including QuickNode) commonly expose standard Ethereum JSON-RPC methods such as `eth_feeHistory` (for fee distributions) and may offer enhanced endpoints/analytics depending on plan. Even without proprietary endpoints, a more robust fee strategy can be built from standard primitives. In other words, “utilities” matter because the current strategy is single-sample and therefore more exposed to outliers.

## Practical conclusions (based on the current implementation)

1. The dominant MEV risk is not “value extraction from swaps,” because the code does not swap or trade. The on-chain action is a stablecoin transfer.
2. The realistic MEV exposure is operational:

   * fee outliers from a simplistic gas price strategy,
   * delayed inclusion under congestion,
   * privacy exposure of payment flows.
3. Nonce handling details (pending vs latest) and any external retry behavior will strongly influence fee and latency variance, even if no adversary is “attacking.”

## Recommendations that follow directly from the code

These suggestions are intentionally scoped to what the code is doing today.

### 1) Improve fee selection to reduce outliers and landing-time variance

The current approach:

```ts
publicClient.getGasPrice()
```

is a single value that may not match current inclusion conditions. A more stable approach is to select fees based on recent blocks (fee history) and choose a percentile appropriate for the target landing time. This reduces both overpaying and stuck transactions. Even if the chain uses legacy behavior, smoothing and bounding gas price selection helps.

### 2) Ensure nonce is derived with pending awareness

If multiple payments can be executed concurrently from the same custodial wallet, use pending nonce semantics to avoid collisions and unintended replacement. This reduces operational exposure where transactions fight each other for inclusion and create fee escalation patterns.

### 3) Consider privacy-aware submission paths for sensitive payments (optional, based on product requirements)

If customer privacy and transaction observability are important, evaluate whether certain payments should be submitted through a private path rather than public broadcast. This is not required to prevent sandwich attacks (since transfers are low-risk), but it can reduce surveillance and targeted behavior. Whether this is worth it depends on business requirements, compliance constraints, and cost.

### 4) Log and measure the variables that explain MEV-like behavior

Even a simple report benefits from concrete metrics. The code already persists timestamps (`submittedAt`, `confirmedAt`) and fee fields (`networkFeeAmount` in `stepsTable`). Ensure the system consistently records:

* broadcast time,
* confirmation time,
* effective paid fee (from receipt),
* any replacement attempts (if they exist),
  so that outliers can be explained rather than treated as noise.

## Closing note

Given the current code path, the MEV story is straightforward: the on-chain payment is a stablecoin transfer, which is structurally resistant to classic trading MEV, but it is still exposed to the inclusion auction and public visibility of mempool/chain. Most improvements therefore come from better fee/nonce engineering and, if desired, privacy-aware broadcast strategies rather than swap-protection mechanisms.
