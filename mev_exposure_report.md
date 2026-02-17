# MEV Exposure Report (Payment Service)

## Scope

This report evaluates MEV exposure for the on-chain payment execution path in `PaymentsService`, based on the implementation of `executePayment()` and `buildUnsignedTransaction()`. The goal is to identify what MEV vectors are realistically applicable given the current transaction behavior, and to state clear, code-based conclusions.

In this report, “MEV exposure” refers to the extent to which a transaction can be observed and then affected by third parties who influence ordering and inclusion (searchers/builders/validators), resulting in value extraction, higher costs, delayed inclusion, or privacy leakage.

## What transaction is being sent on-chain

The on-chain action produced by this service is a single ERC-20 token transfer for USDC or USDT. The transaction data is explicitly encoded as `transfer(to, amount)` and sent to the token contract address.

This is directly shown in `buildUnsignedTransaction()`:

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

The signed transaction is produced via Turnkey and then submitted to the network:

```ts
const unsignedTransaction = await this.buildUnsignedTransaction(
  sourceAccount.cryptoWalletAddress,
  destinationAccount.cryptoWalletAddress,
  BigInt(transferStep.fromAmount),
  payment.fromCurrency || "USDC",
);

signedTransaction = await this.signTransactionWithTurnkey(
  workspace.turnkeySubOrgId,
  sourceAccount.cryptoWalletAddress,
  unsignedTransaction,
  signature,
  payment.fromNetwork as NetworkKey,
);

const txHash = await this.blockchainService.submitSignedTransaction(
  signedTransaction,
  destinationAccount.cryptoWalletAddress,
  BigInt(transferStep.fromAmount),
  payment.fromCurrency || "USDC",
  payment.fromNetwork as NetworkKey,
);
```

This matters because MEV exposure depends heavily on what the transaction *does*. A plain token transfer behaves very differently from price-sensitive DeFi transactions.

## MEV exposure analysis

### 1) Low exposure to “classic” trading MEV (sandwiching/arbitrage)

The most widely discussed MEV attacks (sandwiching swaps, backrunning to capture price impact, extracting arbitrage from AMM trades) require the victim transaction to create a price-relevant state transition. In the code shown here, the service does not interact with an AMM, does not execute a swap, and does not submit a sequence of transactions whose ordering creates a measurable trading opportunity. It submits a single ERC-20 transfer.

Because of that, the typical “value extraction from execution semantics” is low: there is no slippage to worsen, no pool reserves to manipulate, and no quoted execution price to exploit.

The practical conclusion is that the payment execution path is structurally resistant to the most common MEV value-extraction patterns seen in DeFi trading.

### 2) Meaningful exposure to inclusion/ordering dynamics (cost and latency)

Even when a transaction is not economically exploitable via sandwiching, it still participates in an inclusion market: it competes with other pending transactions for space in upcoming blocks. That means it can be delayed, or require a higher fee to achieve timely inclusion, especially during congestion.

In `buildUnsignedTransaction()`, fee selection is based on a single RPC quote:

```ts
const [nonce, gasPrice] = await Promise.all([
  publicClient.getTransactionCount({ address: checksummedAddress }),
  publicClient.getGasPrice(),
]);
```

and the transaction is serialized with `gasPrice` rather than explicit EIP-1559 fee caps:

```ts
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

From an MEV exposure standpoint, the key point is that ordering and inclusion are not purely technical; they are economic. The stronger the fee selection is tied to a single instantaneous value, the more likely the system experiences fee outliers and uncertain landing time under changing network conditions.

This is not “an attacker stealing funds,” but it is still a form of MEV exposure because the same actors and mechanisms that generate MEV (searchers bidding, builders selecting profitable bundles, validators preferring certain flows) shape inclusion outcomes. The impact shows up as:

* higher-than-necessary fees in some cases,
* delayed confirmations in other cases.

### 3) Privacy exposure: payment flows are observable and linkable

A public on-chain ERC-20 transfer reveals the sender, recipient, token contract, amount, and timing. That means payment flows can be monitored and correlated. This matters more for payment systems than for casual transfers, because repeated payments from a custodial wallet can create a clear, linkable behavioral footprint.

Nothing in the shown code suggests an attempt to hide transaction details at broadcast time (for example, private submission pathways). Therefore, assuming standard broadcast behavior behind `submitSignedTransaction`, this flow is exposed to mempool and on-chain observers.

This privacy exposure is a legitimate MEV-related concern because visibility is a prerequisite for many MEV behaviors: if a transaction is visible before inclusion, third parties can react to it. In this specific payment design, the reaction is unlikely to be “profit via sandwich,” but visibility still enables surveillance and targeted inclusion preferences.

### 4) Nonce handling can create operational MEV-like failure modes

The nonce is fetched as:

```ts
publicClient.getTransactionCount({ address: checksummedAddress })
```

If the wallet can have multiple pending transactions, and if the nonce is not derived with pending awareness, it becomes easier to create operational patterns that resemble MEV competition: transactions replacing each other, fee escalation, and inconsistent inclusion. The code shown does not include explicit replacement logic, but nonce selection is foundational; if this is wrong, the system tends to compensate later by retrying and increasing fees.

This is relevant to MEV exposure because it increases the probability that the service “pays the auction” inefficiently or suffers unpredictable inclusion behavior.

## Conclusions

Based on the code provided:

1. The on-chain payment execution is a single ERC-20 `transfer(to, amount)` for USDC/USDT. This makes classic trading MEV (sandwiching/arbitrage around swaps) low likelihood for this path.

2. The realistic MEV exposure is primarily about inclusion and ordering outcomes: fee outliers and confirmation latency variance driven by mempool competition and the fee selection approach.

3. There is meaningful privacy exposure because payment transfers are inherently observable and may be linkable across customers when using a shared custodial wallet.

4. Nonce handling is a secondary but important contributor: if concurrency exists, nonce strategy can amplify cost/latency variance and create replacement patterns that increase exposure to the inclusion auction.

## Notes on evidence and limits

This report is strictly grounded in the behavior visible in `executePayment()` and `buildUnsignedTransaction()`. The final broadcast behavior depends on how `blockchainService.submitSignedTransaction(...)` is implemented (for example, whether it uses standard public RPC submission). The conclusions above assume standard public broadcast semantics, which is consistent with the overall design shown.


