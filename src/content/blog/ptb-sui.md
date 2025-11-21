---
title: 'Programmable Transaction Blocks'
description: "A comprehensive guide to Programmable Transaction Blocks (PTBs) - Sui's revolutionary feature that enables complex, composable, and atomic operations in a single transaction"
pubDate: 'November 21 2025'
heroImage: '../../assets/blog-placeholder-3.jpg'
tags:
  - move
  - sui
  - ptb
---

# Mastering Programmable Transaction Blocks

Programmable Transaction Blocks (PTBs) represent one of Sui's most groundbreaking innovations, fundamentally changing how developers interact with blockchain technology. Unlike traditional blockchains where each operation requires a separate transaction, PTBs allow you to compose up to 1,024 operations into a single atomic execution unit. This capability unlocks unprecedented levels of efficiency, composability, and user experience.

## What are Programmable Transaction Blocks?

A Programmable Transaction Block is a heterogeneous sequence of commands that executes atomically on the Sui blockchain. Think of it as a mini-program that can call multiple smart contracts, manipulate various objects, and perform complex operations - all within a single transaction.

### The Problem PTBs Solve

On traditional blockchains, executing multiple related operations presents significant challenges:

**Sequential Transactions**: Users must sign and execute multiple transactions in sequence, waiting for each to finalize before proceeding to the next. This creates friction and degrades user experience.

**High Gas Costs**: Each individual transaction incurs its own gas fee, making multi-step operations expensive.

**No Atomicity**: If one transaction in a sequence fails, earlier transactions may have already executed, leaving your application in an inconsistent state.

**Smart Contract Workarounds**: Developers are forced to write wrapper contracts just to batch operations together, adding complexity and potential security vulnerabilities.

PTBs elegantly solve all these problems by enabling complex, composable transactions with guaranteed atomicity.

## Key Features of PTBs

### 1. Composability

PTBs can invoke any public function from any deployed smart contract, regardless of whether it's marked as an `entry` function. This means you can chain together calls across multiple protocols seamlessly.

### 2. Atomicity

All operations within a PTB either execute entirely or fail completely. If any single command fails, the entire transaction is rolled back, ensuring your application never enters an inconsistent state.

### 3. High Capacity

Each PTB can contain up to 1,024 individual operations, allowing you to build sophisticated transaction flows that would be impossible or impractical on other blockchains.

### 4. Cost Efficiency

Executing multiple operations in a single PTB is significantly cheaper than executing them as individual transactions. The gas savings can be substantial for complex workflows.

### 5. Flexible Data Flow

The output of one command can be used as input for subsequent commands, enabling powerful composition patterns. This allows you to build complex logic without deploying custom smart contracts.

## Why PTBs are Revolutionary

Consider a DeFi aggregator that finds the best swap rate across multiple protocols. On traditional blockchains, this would require:

1. Approving token spend (Transaction 1)
2. Calling the aggregator contract (Transaction 2)
3. The aggregator finding the best rate and executing the swap (within Transaction 2)

With PTBs on Sui, this entire flow happens in a single transaction with guarantees that you'll either get the displayed price or the transaction reverts completely. No intermediate states, no sandwich attacks, no complexity.

## Setting Up Your PTB Development Environment

Before diving into PTB development, ensure you have the necessary tools:

### Prerequisites

```bash
# Install Sui CLI
cargo install --locked --git https://github.com/MystenLabs/sui.git --branch devnet sui

# Verify installation
sui --version

# Install Sui TypeScript SDK
npm install @mysten/sui
```

### Environment Configuration

```bash
# Connect to Sui devnet
sui client new-env --alias devnet --rpc https://fullnode.devnet.sui.io:443

# Switch to devnet
sui client switch --env devnet

# Get test tokens
sui client faucet
```

## Building Your First PTB with TypeScript SDK

Let's start with a practical example: sending SUI tokens to multiple recipients in a single transaction.

### Basic Setup

```typescript
import { Transaction } from '@mysten/sui/transactions';
import { SuiClient } from '@mysten/sui/client';

// Initialize the Sui client
const client = new SuiClient({ url: 'https://fullnode.devnet.sui.io:443' });

// Create a new transaction
const tx = new Transaction();
```

### Example 1: Simple SUI Transfer

```typescript
// Split 100 MIST from the gas coin
const [coin] = tx.splitCoins(tx.gas, [tx.pure.u64(100)]);

// Transfer to a recipient
tx.transferObjects([coin], tx.pure.address('0xRECIPIENT_ADDRESS'));

// Execute the transaction
const result = await client.signAndExecuteTransaction({
  signer: keypair,
  transaction: tx
});
```

### Example 2: Batch Transfers

This is where PTBs really shine - sending tokens to multiple addresses atomically:

```typescript
interface Transfer {
  to: string;
  amount: number;
}

const transfers: Transfer[] = [
  { to: '0xADDRESS1', amount: 1000 },
  { to: '0xADDRESS2', amount: 2000 },
  { to: '0xADDRESS3', amount: 3000 }
];

const tx = new Transaction();

// Split the gas coin into multiple coins
const coins = tx.splitCoins(
  tx.gas,
  transfers.map(transfer => tx.pure.u64(transfer.amount))
);

// Create transfer commands for each recipient
transfers.forEach((transfer, index) => {
  tx.transferObjects([coins[index]], tx.pure.address(transfer.to));
});

// Execute the batch transfer
await client.signAndExecuteTransaction({
  signer: keypair,
  transaction: tx
});
```

## PTB Commands Reference

PTBs support several core commands that you can chain together:

### splitCoins

Creates new coins with specified amounts from a source coin.

```typescript
const [coin1, coin2] = tx.splitCoins(tx.gas, [
  tx.pure.u64(100),
  tx.pure.u64(200)
]);
```

### mergeCoins

Combines multiple coins into a single coin.

```typescript
tx.mergeCoins(
  tx.object(destinationCoin),
  [tx.object(coin1), tx.object(coin2)]
);
```

### transferObjects

Transfers objects to a specified address.

```typescript
tx.transferObjects(
  [tx.object(nft1), tx.object(nft2)],
  tx.pure.address(recipientAddress)
);
```

### moveCall

Invokes a Move function on a smart contract.

```typescript
const result = tx.moveCall({
  target: '0xPACKAGE::module::function',
  arguments: [
    tx.pure.string('Hello'),
    tx.pure.u64(42),
    tx.object(someObject)
  ],
  typeArguments: ['0x2::sui::SUI']
});
```

### makeMoveVec

Creates a vector of objects for passing to Move functions.

```typescript
const nftVector = tx.makeMoveVec({
  elements: [tx.object(nft1), tx.object(nft2), tx.object(nft3)]
});

tx.moveCall({
  target: '0xPACKAGE::gallery::add_nfts',
  arguments: [tx.object(gallery), nftVector]
});
```

## Advanced PTB Patterns

### Pattern 1: Chaining Transaction Results

One of PTB's most powerful features is the ability to use the output of one command as input for another:

```typescript
const tx = new Transaction();

// Mint an NFT
const [nft] = tx.moveCall({
  target: '0xPACKAGE::nft::mint',
  arguments: [
    tx.pure.string('Cool NFT'),
    tx.pure.string('An awesome NFT'),
    tx.pure.string('https://example.com/nft.png')
  ]
});

// Immediately list it on a marketplace
tx.moveCall({
  target: '0xMARKETPLACE::market::list',
  arguments: [
    nft,  // Use the freshly minted NFT
    tx.pure.u64(1000000),  // Price: 1 SUI
    tx.object(marketplaceObject)
  ]
});
```

### Pattern 2: DeFi Composition

Execute a complex DeFi strategy atomically:

```typescript
const tx = new Transaction();

// 1. Split coins for the operation
const [depositCoin] = tx.splitCoins(tx.gas, [tx.pure.u64(10000000)]);

// 2. Deposit into a lending protocol
const [receipt] = tx.moveCall({
  target: '0xLENDING::pool::deposit',
  arguments: [tx.object(pool), depositCoin]
});

// 3. Use the receipt as collateral to borrow
const [borrowedCoin] = tx.moveCall({
  target: '0xLENDING::pool::borrow',
  arguments: [tx.object(pool), receipt, tx.pure.u64(5000000)]
});

// 4. Swap the borrowed funds on a DEX
const [swappedCoin] = tx.moveCall({
  target: '0xDEX::swap::swap_exact_input',
  arguments: [
    tx.object(dexPool),
    borrowedCoin,
    tx.pure.u64(4500000)  // Min output
  ],
  typeArguments: ['0x2::sui::SUI', '0xUSDC::usdc::USDC']
});

// 5. Transfer final result to user
tx.transferObjects([swappedCoin], tx.pure.address(userAddress));
```

### Pattern 3: NFT Batch Operations

Mint and distribute multiple NFTs efficiently:

```typescript
const tx = new Transaction();
const recipients = ['0xADDR1', '0xADDR2', '0xADDR3'];

// Mint multiple NFTs in one go
const nfts = tx.moveCall({
  target: '0xNFT::collection::mint_batch',
  arguments: [
    tx.object(collection),
    tx.pure.u64(recipients.length)
  ]
});

// Distribute to recipients
recipients.forEach((recipient, index) => {
  tx.transferObjects([nfts[index]], tx.pure.address(recipient));
});
```

## Working with the Gas Coin

PTBs introduce a special `tx.gas` reference that represents the gas payment coin. This enables elegant patterns for coin management:

### Using Gas Coin for Payments

```typescript
// Split from gas coin - no need to select coins beforehand
const [paymentCoin] = tx.splitCoins(tx.gas, [tx.pure.u64(1000000)]);

// Use it immediately
tx.moveCall({
  target: '0xPACKAGE::shop::purchase',
  arguments: [tx.object(item), paymentCoin]
});
```

### Merging into Gas Coin

```typescript
// Merge other coins back into gas coin
tx.mergeCoins(tx.gas, [
  tx.object(coin1),
  tx.object(coin2)
]);
```

### Transferring All Funds

```typescript
// Transfer the entire gas coin (all funds)
tx.transferObjects([tx.gas], tx.pure.address(recipientAddress));
```

## Building PTBs with Sui CLI

For developers who prefer the command line or need to write bash scripts, Sui provides a powerful CLI for PTB construction:

### Basic CLI PTB

```bash
sui client ptb \
  --split-coins gas [1000,2000,3000] \
  --assign coins \
  --transfer-objects [coins.0,coins.1,coins.2] @0xRECIPIENT \
  --gas-budget 5000000
```

### CLI PTB with Move Calls

```bash
sui client ptb \
  --move-call $PACKAGE::nft::mint '"My NFT"' '"Description"' '"https://img.url"' \
  --assign nft \
  --transfer-objects [nft] @0xRECIPIENT \
  --gas-budget 10000000
```

### Preview Mode

Use `--preview` to see what your PTB will do without executing it:

```bash
sui client ptb \
  --split-coins gas [1000,2000] \
  --assign coins \
  --transfer-objects [coins.0,coins.1] @0xRECIPIENT \
  --gas-budget 5000000 \
  --preview
```

### Using Variables

```bash
sui client ptb \
  --assign recipient @0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb1 \
  --assign amount 1000000 \
  --split-coins gas [amount] \
  --assign coin \
  --transfer-objects [coin] recipient \
  --gas-budget 5000000
```

## Input Types and Values

PTBs support two types of inputs: objects and pure values.

### Object Inputs

Reference on-chain objects by their ID:

```typescript
// Owned or immutable object
tx.object('0xOBJECT_ID');

// Shared object (requires initial shared version)
tx.object(Inputs.SharedObjectRef({
  objectId: '0xOBJECT_ID',
  initialSharedVersion: '123',
  mutable: true
}));
```

### Pure Value Inputs

Pass serialized data to Move functions:

```typescript
// New convenient syntax
tx.pure.u64(100);
tx.pure.u8(255);
tx.pure.bool(true);
tx.pure.string('Hello, Sui!');
tx.pure.address('0xADDRESS');
tx.pure.vector('u64', [1, 2, 3, 4, 5]);

// Raw bytes
tx.pure(new Uint8Array([1, 2, 3, 4]));
```

## Gas Configuration

PTBs provide fine-grained control over gas parameters:

### Setting Gas Price

```typescript
tx.setGasPrice(1000); // In MIST
```

### Setting Gas Budget

```typescript
tx.setGasBudget(10000000); // In MIST (0.01 SUI)
```

### Custom Gas Payment

```typescript
tx.setGasPayment([
  { objectId: '0xCOIN1', version: '1', digest: 'DIGEST1' },
  { objectId: '0xCOIN2', version: '1', digest: 'DIGEST2' }
]);
```

### Auto Gas Configuration

By default, the SDK handles gas automatically:
- Fetches current network gas price
- Performs a dry-run to estimate gas consumption
- Selects appropriate gas coins from your wallet

## Sponsored Transactions

PTBs support transaction sponsorship, where a third party pays gas fees:

```typescript
// Build the transaction kind without gas data
const tx = new Transaction();
// ... add operations ...

const kindBytes = await tx.build({
  client,
  onlyTransactionKind: true
});

// Sponsor constructs the full transaction
const sponsoredTx = Transaction.fromKind(kindBytes);
sponsoredTx.setSender(userAddress);
sponsoredTx.setGasOwner(sponsorAddress);
sponsoredTx.setGasPayment(sponsorCoins);

// Sponsor signs and executes
await client.signAndExecuteTransaction({
  signer: sponsorKeypair,
  transaction: sponsoredTx
});
```

This enables gasless user experiences - crucial for mainstream adoption.

## Building Offline PTBs

For enhanced security or air-gapped signing, build PTBs without network access:

```typescript
import { Inputs } from '@mysten/sui/transactions';

const tx = new Transaction();

// Use fully specified inputs
tx.object(Inputs.ObjectRef({
  objectId: '0xOBJECT_ID',
  version: '123',
  digest: 'DIGEST_HASH'
}));

// Provide pre-serialized pure values
tx.pure(bcsSerializedBytes);

// Set all gas parameters
tx.setGasPrice(1000);
tx.setGasBudget(10000000);
tx.setGasPayment([{ objectId, version, digest }]);
tx.setSender(senderAddress);

// Build without provider
const bytes = await tx.build({ client: null });
```

## Wallet Integration

PTBs integrate seamlessly with wallets through the Wallet Standard:

### dApp Side

```typescript
import { Transaction } from '@mysten/sui/transactions';

// Build your PTB
const tx = new Transaction();
// ... add operations ...

// Request signature from wallet
const result = await wallet.signAndExecuteTransaction({
  transaction: tx,
  chain: 'sui:mainnet'
});
```

### Wallet Side

```typescript
// Serialize for cross-context communication
function handleSignRequest(tx: Transaction) {
  const serialized = tx.serialize();
  // Send to wallet UI
  sendToWalletUI(serialized);
}

// Deserialize in wallet context
function processInWallet(serialized: string) {
  const tx = Transaction.from(serialized);
  // Wallet can now inspect, modify gas, and sign
}
```

## Error Handling and Debugging

### Dry Run for Testing

Always test PTBs before executing:

```typescript
const dryRunResult = await client.dryRunTransactionBlock({
  transactionBlock: await tx.build({ client })
});

if (dryRunResult.effects.status.status === 'failure') {
  console.error('Transaction would fail:', dryRunResult.effects.status.error);
}
```

### Common Errors

**Non-Drop Values**: If a Move call returns a non-droppable value, it must be consumed within the PTB:

```typescript
// ❌ Bad - leaves non-drop value unused
tx.moveCall({
  target: '0xPACKAGE::module::create_thing'
});

// ✅ Good - transfers or destroys the value
const [thing] = tx.moveCall({
  target: '0xPACKAGE::module::create_thing'
});
tx.transferObjects([thing], tx.pure.address(userAddress));
```

**Object Ownership**: Ensure you have access to objects you're trying to use:

```typescript
// Objects must be owned by the transaction sender
// or be shared/immutable objects
```

**Gas Exhaustion**: Set adequate gas budget for complex operations:

```typescript
// For complex PTBs with multiple Move calls
tx.setGasBudget(50000000); // 0.05 SUI
```

## PTB Best Practices

### 1. Optimize Gas Usage

- Group related operations into single PTBs
- Use `tx.gas` for simple payments instead of selecting coins
- Avoid unnecessary coin splits and merges

### 2. Handle Errors Gracefully

```typescript
try {
  const result = await client.signAndExecuteTransaction({
    signer: keypair,
    transaction: tx
  });
  
  if (result.effects?.status?.status === 'failure') {
    // Handle transaction failure
    console.error('Transaction failed:', result.effects.status.error);
  }
} catch (error) {
  // Handle execution error
  console.error('Execution error:', error);
}
```

### 3. Use Descriptive Variable Names

```typescript
// ✅ Good
const [depositReceipt] = tx.moveCall({ target: '0xPOOL::deposit' });
const [borrowedFunds] = tx.moveCall({ target: '0xPOOL::borrow' });

// ❌ Bad
const [a] = tx.moveCall({ target: '0xPOOL::deposit' });
const [b] = tx.moveCall({ target: '0xPOOL::borrow' });
```

### 4. Validate Inputs

```typescript
function createTransferPTB(recipients: string[], amounts: number[]) {
  if (recipients.length !== amounts.length) {
    throw new Error('Recipients and amounts length mismatch');
  }
  
  if (amounts.some(a => a <= 0)) {
    throw new Error('All amounts must be positive');
  }
  
  const tx = new Transaction();
  // ... build PTB
  return tx;
}
```

### 5. Use TypeScript for Type Safety

```typescript
interface SwapParams {
  poolId: string;
  amountIn: number;
  minAmountOut: number;
  tokenIn: string;
  tokenOut: string;
}

function createSwapPTB(params: SwapParams): Transaction {
  const tx = new Transaction();
  
  const [inputCoin] = tx.splitCoins(tx.gas, [tx.pure.u64(params.amountIn)]);
  
  tx.moveCall({
    target: '0xDEX::swap::swap_exact_input',
    arguments: [
      tx.object(params.poolId),
      inputCoin,
      tx.pure.u64(params.minAmountOut)
    ],
    typeArguments: [params.tokenIn, params.tokenOut]
  });
  
  return tx;
}
```

## Real-World Use Cases

### Use Case 1: DeFi Aggregator

```typescript
async function executeOptimalSwap(
  client: SuiClient,
  amount: number,
  tokenIn: string,
  tokenOut: string
) {
  const tx = new Transaction();
  
  // Find best route (off-chain calculation)
  const route = await findBestRoute(amount, tokenIn, tokenOut);
  
  // Split initial amount
  const [currentCoin] = tx.splitCoins(tx.gas, [tx.pure.u64(amount)]);
  
  // Execute swaps through multiple DEXs
  for (const hop of route.hops) {
    const [outputCoin] = tx.moveCall({
      target: `${hop.dex}::swap::swap`,
      arguments: [
        tx.object(hop.poolId),
        currentCoin,
        tx.pure.u64(hop.minOutput)
      ],
      typeArguments: [hop.tokenIn, hop.tokenOut]
    });
    currentCoin = outputCoin;
  }
  
  // Transfer final output
  tx.transferObjects([currentCoin], tx.pure.address(userAddress));
  
  return client.signAndExecuteTransaction({
    signer: keypair,
    transaction: tx
  });
}
```

### Use Case 2: NFT Marketplace

```typescript
async function buyAndListNFT(
  client: SuiClient,
  marketplaceId: string,
  listingId: string,
  resellPrice: number
) {
  const tx = new Transaction();
  
  // Purchase NFT from marketplace
  const [nft, changeCoin] = tx.moveCall({
    target: '0xMARKET::marketplace::buy',
    arguments: [
      tx.object(marketplaceId),
      tx.object(listingId),
      tx.gas
    ]
  });
  
  // Immediately relist at higher price
  tx.moveCall({
    target: '0xMARKET::marketplace::list',
    arguments: [
      tx.object(marketplaceId),
      nft,
      tx.pure.u64(resellPrice)
    ]
  });
  
  // Merge change back to gas
  tx.mergeCoins(tx.gas, [changeCoin]);
  
  return client.signAndExecuteTransaction({
    signer: keypair,
    transaction: tx
  });
}
```

### Use Case 3: Gaming - Batch Rewards Distribution

```typescript
async function distributeGameRewards(
  client: SuiClient,
  winners: Array<{ address: string; nftIds: string[]; coinReward: number }>
) {
  const tx = new Transaction();
  
  // Split coins for all winners
  const coinRewards = tx.splitCoins(
    tx.gas,
    winners.map(w => tx.pure.u64(w.coinReward))
  );
  
  // Distribute to each winner
  winners.forEach((winner, index) => {
    // Transfer coin reward
    tx.transferObjects([coinRewards[index]], tx.pure.address(winner.address));
    
    // Transfer NFT rewards
    const nfts = winner.nftIds.map(id => tx.object(id));
    tx.transferObjects(nfts, tx.pure.address(winner.address));
  });
  
  return client.signAndExecuteTransaction({
    signer: keypair,
    transaction: tx
  });
}
```

## Performance Optimization

### Batch Operations Efficiently

```typescript
// ❌ Less efficient - multiple transactions
for (const recipient of recipients) {
  const tx = new Transaction();
  const [coin] = tx.splitCoins(tx.gas, [tx.pure.u64(amount)]);
  tx.transferObjects([coin], tx.pure.address(recipient));
  await client.signAndExecuteTransaction({ signer: keypair, transaction: tx });
}

// ✅ Optimal - single PTB
const tx = new Transaction();
const coins = tx.splitCoins(
  tx.gas,
  recipients.map(() => tx.pure.u64(amount))
);
recipients.forEach((recipient, i) => {
  tx.transferObjects([coins[i]], tx.pure.address(recipient));
});
await client.signAndExecuteTransaction({ signer: keypair, transaction: tx });
```

### Minimize Object Reads

```typescript
// Read shared objects once and reuse the reference
const poolRef = tx.object(poolId);

tx.moveCall({
  target: '0xDEX::pool::add_liquidity',
  arguments: [poolRef, coin1, coin2]
});

tx.moveCall({
  target: '0xDEX::pool::claim_fees',
  arguments: [poolRef]
});
```

## Testing PTBs

### Unit Testing with Mock Client

```typescript
import { describe, it, expect } from 'vitest';

describe('PTB Tests', () => {
  it('should create valid transfer PTB', async () => {
    const tx = new Transaction();
    const [coin] = tx.splitCoins(tx.gas, [tx.pure.u64(1000)]);
    tx.transferObjects([coin], tx.pure.address('0xRECIPIENT'));
    
    // Validate PTB structure
    const built = await tx.build({ client: mockClient });
    expect(built).toBeDefined();
  });
  
  it('should handle multiple operations', async () => {
    const tx = createComplexPTB();
    const dryRun = await client.dryRunTransactionBlock({
      transactionBlock: await tx.build({ client })
    });
    
    expect(dryRun.effects.status.status).toBe('success');
  });
});
```

## Common Patterns Library

### Pattern: Flash Loan

```typescript
function createFlashLoanPTB(
  lendingPool: string,
  amount: number,
  operations: (borrowedCoin: any) => void
): Transaction {
  const tx = new Transaction();
  
  // Borrow
  const [borrowedCoin, receipt] = tx.moveCall({
    target: '0xLENDING::pool::flash_loan',
    arguments: [tx.object(lendingPool), tx.pure.u64(amount)]
  });
  
  // Execute custom operations with borrowed funds
  operations(borrowedCoin);
  
  // Repay
  tx.moveCall({
    target: '0xLENDING::pool::repay_flash_loan',
    arguments: [tx.object(lendingPool), borrowedCoin, receipt]
  });
  
  return tx;
}
```

### Pattern: Approve and Call

```typescript
function approveAndSwap(
  tokenObject: string,
  spender: string,
  swapParams: SwapParams
): Transaction {
  const tx = new Transaction();
  
  // No explicit approval needed - just use the token
  // Sui's object ownership model handles this elegantly
  
  tx.moveCall({
    target: `${spender}::swap::execute`,
    arguments: [
      tx.object(tokenObject),
      tx.pure.u64(swapParams.minOutput)
    ]
  });
  
  return tx;
}
```

## Security Considerations

### 1. Validate Move Call Targets

```typescript
const TRUSTED_PACKAGES = new Set([
  '0x2', // Sui Framework
  '0x3', // Sui System
  '0xYOUR_VERIFIED_PACKAGE'
]);

function validateTarget(target: string) {
  const packageId = target.split('::')[0];
  if (!TRUSTED_PACKAGES.has(packageId)) {
    throw new Error(`Untrusted package: ${packageId}`);
  }
}
```

### 2. Check Dry Run Results

```typescript
async function safeExecute(tx: Transaction) {
  const dryRun = await client.dryRunTransactionBlock({
    transactionBlock: await tx.build({ client })
  });
  
  if (dryRun.effects.status.status !== 'success') {
    throw new Error('Dry run failed');
  }
  
  // Verify expected object changes
  const objectChanges = dryRun.effects.mutated?.length || 0;
  if (objectChanges > EXPECTED_CHANGES) {
    throw new Error('Unexpected object mutations');
  }
  
  return client.signAndExecuteTransaction({ signer: keypair, transaction: tx });
}
```

### 3. Implement Slippage Protection

```typescript
function createSwapWithSlippage(
  amount: number,
  expectedOutput: number,
  maxSlippage: number // percentage
): Transaction {
  const minOutput = expectedOutput * (1 - maxSlippage / 100);
  
  const tx = new Transaction();
  const [inputCoin] = tx.splitCoins(tx.gas, [tx.pure.u64(amount)]);
  
  tx.moveCall({
    target: '0xDEX::swap::swap',
    arguments: [
      tx.object(poolId),
      inputCoin,
      tx.pure.u64(Math.floor(minOutput)) // Minimum acceptable output
    ]
  });
  
  return tx;
}
```

## Monitoring and Analytics

### Track PTB Execution

```typescript
async function executeWithMonitoring(tx: Transaction) {
  const startTime = Date.now();
  
  try {
    const result = await client.signAndExecuteTransaction({
      signer: keypair,
      transaction: tx,
      options: {
        showEffects: true,
        showObjectChanges: true
      }
    });
    
    const executionTime = Date.now() - startTime;
    
    // Log metrics
    console.log({
      digest: result.digest,
      status: result.effects?.status?.status,
      gasUsed: result.effects?.gasUsed,
      executionTime,
      objectsChanged: result.objectChanges?.length
    });
    
    return result;
  } catch (error) {
    console.error('PTB execution failed', { error, executionTime: Date.now() - startTime });
    throw error;
  }
}
```

## Migration from Legacy Transactions

If you're migrating from older Sui transaction types:

```typescript
// Old: PaySui
sui.paySui(recipients, amounts, gas);

// New: PTB
const tx = new Transaction();
const coins = tx.splitCoins(tx.gas, amounts.map(a => tx.pure.u64(a)));
recipients.forEach((r, i) => tx.transferObjects([coins[i]], tx.pure.address(r)));

// Old: MergeCoin
sui.mergeCoin(target, sources);

// New: PTB
const tx = new Transaction();
tx.mergeCoins(tx.object(target), sources.map(s => tx.object(s)));
```

## What's Next?

Now that you understand PTBs, explore these advanced topics:

1. **Optimize Gas Costs**: Learn strategies for minimizing gas consumption in complex PTBs
2. **Build DeFi Protocols**: Create sophisticated financial applications leveraging PTB composability
3. **Implement Gasless Transactions**: Set up sponsored transactions for seamless UX
4. **Create Transaction Builders**: Build reusable PTB templates for your dApp
5. **Explore Kiosk Standards**: Combine PTBs with Sui's Kiosk framework for advanced NFT functionality

## Resources

- [Official PTB Documentation](https://docs.sui.io/concepts/transactions/prog-txn-blocks)
- [Sui TypeScript SDK](https://sdk.mystenlabs.com/typescript)
- [PTB Examples Repository](https://github.com/MystenLabs/sui/tree/main/sdk/typescript/test/e2e)
- [Sui Move Intro Course](https://github.com/sui-foundation/sui-move-intro-course)
- [Sui Discord Community](https://discord.gg/sui)

## Conclusion

Programmable Transaction Blocks represent a paradigm shift in blockchain development. By enabling complex, atomic, and composable operations, PTBs unlock use cases that were previously impossible or impractical on traditional blockchains.

The combination of up to 1,024 operations per transaction, guaranteed atomicity, and the ability to chain outputs as inputs creates a powerful programming model. Whether you're building DeFi protocols, NFT marketplaces, gaming applications, or any other decentralized application, mastering PTBs is essential for creating efficient, user-friendly experiences on Sui.

Start experimenting with PTBs today, and you'll quickly discover why Sui developers consider them one of the blockchain's most revolutionary features. The future of decentralized applications is programmable, composable, and atomic - and it's already here on Sui.

---

*Ready to level up? Check out our upcoming posts on advanced PTB patterns, building multi-protocol DeFi aggregators, and optimizing gas costs for production applications.*
