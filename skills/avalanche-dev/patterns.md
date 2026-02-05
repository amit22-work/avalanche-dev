# patterns.md

## Purpose of This Document

This document defines approved, security-reviewed implementation patterns for Avalanche C-Chain development. These patterns have been validated against the threat model and security requirements defined in security.md.

**Deviation from these patterns increases risk and is discouraged.** These patterns are mandatory for production deployments. When in doubt, default to the most restrictive pattern.

## Pattern 1: Avalanche Network & Chain Validation

Chain ID validation is the first line of defense against cross-chain attacks and user error. All interactions must validate the target chain before proceeding.

### Requirements

- Explicit chainId validation before any blockchain logic
- Refusal behavior if chainId ≠ 43114 (Mainnet) or 43113 (Fuji Testnet)
- Never auto-switch chains without explicit user approval
- User-initiated chain switching only
- Chain ID must be validated on every transaction

### Implementation

```typescript
import { createPublicClient, http } from 'viem';
import { avalanche, avalancheFuji } from 'viem/chains';

// Validate chain before any operations
async function validateChain(chainId: number): Promise<boolean> {
  const allowedChains = [43114, 43113]; // Mainnet, Fuji
  if (!allowedChains.includes(chainId)) {
    throw new Error(`Invalid chain ID: ${chainId}. Only Avalanche Mainnet (43114) and Fuji (43113) are allowed.`);
  }
  return true;
}

// Create client with explicit chain validation
const publicClient = createPublicClient({
  chain: avalanche, // or avalancheFuji
  transport: http()
});

// Before any operation, validate chain
const chainId = await publicClient.getChainId();
await validateChain(chainId);
```

### Refusal Behavior

If chainId validation fails:
- Abort all operations immediately
- Report the mismatch to the user
- Do not proceed with any transaction logic
- Require explicit user confirmation before retry

## Pattern 2: viem Client Separation (MANDATORY)

Strict separation between read and write operations is non-negotiable. Mixing client responsibilities creates security risks and violates the principle of least privilege.

### Requirements

- PublicClient = read-only operations exclusively
- WalletClient = signing and transaction submission exclusively
- Never mix responsibilities in a single client instance
- Never reuse a WalletClient for read operations
- Maintain separate client instances for read and write

### Implementation

```typescript
import { createPublicClient, createWalletClient, http } from 'viem';
import { avalanche } from 'viem/chains';
import { walletConnect } from 'viem/connectors';

// Read-only client (PublicClient)
const publicClient = createPublicClient({
  chain: avalanche,
  transport: http('https://api.avax.network/ext/bc/C/rpc')
});

// Write-only client (WalletClient)
const walletClient = createWalletClient({
  chain: avalanche,
  transport: walletConnect({
    // Wallet connection configuration
  })
});

// CORRECT: Use PublicClient for reads
const balance = await publicClient.getBalance({ address: '0x...' });

// CORRECT: Use WalletClient for writes
const hash = await walletClient.writeContract({ /* ... */ });

// FORBIDDEN: Never use WalletClient for reads
// const balance = await walletClient.getBalance({ address: '0x...' }); // WRONG

// FORBIDDEN: Never use PublicClient for writes
// const hash = await publicClient.writeContract({ /* ... */ }); // WRONG
```

### Why Separation Matters

- Read operations do not require signing authority
- Write operations require explicit user approval
- Mixing responsibilities creates confusion about operation safety
- Separation enforces the principle of least privilege
- Separation prevents accidental transaction submission

## Pattern 3: Wallet Connection Pattern

Wallet connections must be explicit, user-initiated, and secure. No automatic connections or silent approvals are permitted.

### Supported Wallets

- **Core Wallet**: First-class support for Avalanche-native wallet
- **WalletConnect v2**: Standard protocol for wallet connections
- **MetaMask**: Allowed only via explicit user choice or WalletConnect v2 connection

### Rules

- No auto-connect: connection requires explicit user action
- No silent connection: user must actively initiate connection
- No implicit approval: connection does not grant transaction approval
- Chain validation required after connection
- User must see connection status clearly

### Connection Flow

```
User → Connect Wallet → Validate Chain → Read-only enabled → Explicit write permission required
```

### Implementation Pattern

```typescript
import { createWalletClient, createPublicClient } from 'viem';
import { avalanche } from 'viem/chains';

// Step 1: User initiates connection (explicit action required)
async function connectWallet() {
  // User must explicitly trigger this function
  // No automatic connection on page load
  const walletClient = createWalletClient({
    chain: avalanche,
    transport: /* wallet transport */
  });
  
  // Step 2: Validate chain after connection
  const chainId = await walletClient.getChainId();
  if (chainId !== 43114 && chainId !== 43113) {
    throw new Error('Invalid chain. Please switch to Avalanche C-Chain.');
  }
  
  // Step 3: Read-only operations enabled
  const publicClient = createPublicClient({
    chain: avalanche,
    transport: http()
  });
  
  // Step 4: Write operations require explicit user approval per transaction
  return { walletClient, publicClient };
}
```

### Forbidden Patterns

- Auto-connecting wallets on page load
- Silently connecting without user action
- Assuming connection grants transaction approval
- Bypassing chain validation after connection

## Pattern 4: Read-Only Contract Interaction

Read operations are safe and do not require user approval. However, RPC providers cannot be trusted blindly, and errors must be handled safely.

### Requirements

- Use PublicClient exclusively for read operations
- Handle RPC errors safely
- Never assume RPC correctness blindly
- Validate read results when possible
- Fail closed on read errors

### Implementation

```typescript
import { createPublicClient, http } from 'viem';
import { avalanche } from 'viem/chains';

const publicClient = createPublicClient({
  chain: avalanche,
  transport: http('https://api.avax.network/ext/bc/C/rpc')
});

// Safe read operation with error handling
async function readContractSafely(
  address: `0x${string}`,
  abi: any,
  functionName: string,
  args: any[]
) {
  try {
    const result = await publicClient.readContract({
      address,
      abi,
      functionName,
      args
    });
    
    // Validate result if possible
    if (result === null || result === undefined) {
      throw new Error('Invalid read result from contract');
    }
    
    return result;
  } catch (error) {
    // Fail closed: do not proceed with invalid data
    console.error('Read operation failed:', error);
    throw new Error('Failed to read contract. RPC may be unreliable.');
  }
}
```

### Error Handling Guidance

- Never proceed with null or undefined read results
- Always validate read results when possible
- Report RPC errors to the user
- Do not retry automatically without user acknowledgment
- Treat RPC failures as potential security issues

## Pattern 5: Transaction Lifecycle (CRITICAL)

The transaction lifecycle is the most critical pattern. Deviation from this flow creates security risks and may result in fund loss.

### Mandatory Transaction Flow

1. **User intent confirmed**: User explicitly requests transaction
2. **Chain validated**: Verify chainId is 43114 or 43113
3. **Transaction simulated**: simulateContract before submission
4. **Simulation result explained**: User sees what will happen
5. **User explicitly approves**: User confirms understanding and intent
6. **Wallet signs**: User signs via wallet (Core Wallet / WalletConnect)
7. **Transaction submitted**: writeContract after approval
8. **Result monitored**: Track transaction status and report to user

### Implementation

```typescript
import { createWalletClient, createPublicClient } from 'viem';
import { avalanche } from 'viem/chains';

async function executeTransactionSafely(
  walletClient: WalletClient,
  publicClient: PublicClient,
  contractAddress: `0x${string}`,
  abi: any,
  functionName: string,
  args: any[]
) {
  // Step 1: User intent confirmed (assumed at function call)
  
  // Step 2: Chain validation
  const chainId = await walletClient.getChainId();
  if (chainId !== 43114 && chainId !== 43113) {
    throw new Error('Invalid chain. Transaction aborted.');
  }
  
  // Step 3: Transaction simulation (MANDATORY)
  let simulation;
  try {
    simulation = await publicClient.simulateContract({
      address: contractAddress,
      abi,
      functionName,
      args,
      account: await walletClient.getAddresses()[0]
    });
  } catch (error) {
    // Step 4: Explain simulation failure
    throw new Error(`Simulation failed: ${error.message}. Transaction will not be sent.`);
  }
  
  // Step 4: Explain simulation result to user
  // Display: function name, contract address, gas estimate, value, chain ID
  const transactionDetails = {
    function: functionName,
    contract: contractAddress,
    gasEstimate: simulation.request.gas,
    value: simulation.request.value || 0n,
    chainId: chainId
  };
  
  // Step 5: User explicitly approves (user action required)
  // User must confirm transaction details before proceeding
  
  // Step 6: Wallet signs (user action via wallet)
  // Step 7: Transaction submitted
  const hash = await walletClient.writeContract({
    address: contractAddress,
    abi,
    functionName,
    args,
    account: await walletClient.getAddresses()[0]
  });
  
  // Step 8: Monitor result
  const receipt = await publicClient.waitForTransactionReceipt({ hash });
  return receipt;
}
```

### Refusal Behavior on Failed Simulation

If simulation fails:
- Transaction MUST NOT be sent
- Error MUST be reported to user
- User MUST acknowledge error before any retry
- No automatic retry without user approval

## Pattern 6: Token Approval Safety Pattern

Token approvals are high-risk operations that can result in unlimited token draining. This pattern minimizes approval risks.

### Why Approvals Are Dangerous

- Infinite approvals grant unlimited spending authority
- Malicious contracts can drain approved tokens
- Approvals persist until explicitly revoked
- Users may forget about existing approvals
- Approval revocation does not prevent already-approved transfers

### Requirements

- Infinite approvals discouraged
- Finite approvals preferred
- Mandatory user warning before approvals
- Approval amount must be visible to user
- Approval recipient must be clearly identified

### Approval Checklist

Before any token approval, the user MUST be informed of:
- Token address and symbol
- Contract address receiving approval
- Approval amount (finite or infinite)
- Risk of approval (token draining possibility)
- How to revoke approval if needed
- Current approval status (if any exists)

### Warning Language Examples

**For Infinite Approvals:**
"WARNING: You are about to grant INFINITE approval to [contract address]. This contract can transfer unlimited [token symbol] from your wallet at any time. Only approve if you fully trust this contract. You can revoke this approval later, but already-approved transfers cannot be prevented."

**For Finite Approvals:**
"WARNING: You are about to approve [amount] [token symbol] to [contract address]. This contract can transfer up to this amount from your wallet. Only approve if you fully trust this contract. You can revoke this approval later, but already-approved transfers cannot be prevented."

### Implementation Pattern

```typescript
async function requestTokenApproval(
  tokenAddress: `0x${string}`,
  spenderAddress: `0x${string}`,
  amount: bigint,
  isInfinite: boolean
) {
  // Display warning with all required information
  const warning = isInfinite
    ? `WARNING: Infinite approval to ${spenderAddress}`
    : `WARNING: Approval of ${amount} tokens to ${spenderAddress}`;
  
  // User must explicitly confirm understanding
  // User approval required before proceeding
  
  // Proceed with approval only after explicit user confirmation
  // Follow Pattern 5: Transaction Lifecycle
}
```

## Pattern 7: Error Handling & Safe Failure

Errors must be handled safely with fail-closed behavior. Never proceed after partial failures or assume operations succeeded without verification.

### Requirements

- Fail closed, not open
- Never retry automatically
- Always surface errors to user
- Never proceed after partial failure
- Require user acknowledgment before retry

### Common Failure Cases

- RPC connection failures
- Transaction simulation failures
- Chain ID mismatches
- Insufficient gas
- Contract revert errors
- Network timeouts
- Invalid contract addresses

### Implementation Pattern

```typescript
async function handleOperationSafely(operation: () => Promise<any>) {
  try {
    const result = await operation();
    
    // Validate result
    if (!result) {
      throw new Error('Operation returned invalid result');
    }
    
    return result;
  } catch (error) {
    // Fail closed: do not proceed
    console.error('Operation failed:', error);
    
    // Surface error to user
    const errorMessage = `Operation failed: ${error.message}. Please review and try again.`;
    
    // Require user acknowledgment before retry
    // Do not retry automatically
    
    throw new Error(errorMessage);
  }
}
```

### Required User Acknowledgment

After any error:
- Error must be clearly explained to user
- User must acknowledge understanding
- User must explicitly request retry
- No automatic retry without user approval

## Pattern 8: Testnet → Mainnet Promotion Pattern

Mainnet deployment is a final, high-risk step. This pattern ensures safe progression from testnet to mainnet.

### Requirements

- Fuji testnet as default for all development
- Fork testing requirement before mainnet use
- Mainnet as final step requiring explicit confirmation
- Explicit confirmation language required for mainnet operations

### Decision Checklist Before Mainnet Usage

Before any mainnet operation, verify:
- [ ] All functionality tested on Fuji testnet
- [ ] Fork testing completed on mainnet fork
- [ ] Security review completed
- [ ] Contract addresses verified
- [ ] Gas costs validated
- [ ] User explicitly confirms mainnet intent
- [ ] User understands irreversible nature of mainnet
- [ ] User understands real funds are at risk

### Implementation Pattern

```typescript
async function promoteToMainnet(
  operation: () => Promise<any>,
  userConfirmation: boolean
) {
  // Default to testnet
  const defaultChain = avalancheFuji; // Chain ID 43113
  
  // Mainnet requires explicit confirmation
  if (chainId === 43114) { // Mainnet
    if (!userConfirmation) {
      throw new Error(
        'Mainnet operation requires explicit user confirmation. ' +
        'This operation is irreversible and uses real funds. ' +
        'Please confirm your intent to proceed with mainnet.'
      );
    }
    
    // Additional validation for mainnet
    // Verify all testnet testing completed
    // Verify fork testing completed
    // Verify security review completed
  }
  
  // Proceed with operation
  return await operation();
}
```

### Explicit Confirmation Language

For mainnet operations, require explicit confirmation:
"This operation will execute on Avalanche Mainnet (Chain ID 43114) using real funds. This action is irreversible. Please confirm you understand the risks and intend to proceed."

## Forbidden Patterns (Explicit)

The following patterns MUST NOT be used. Violation of these prohibitions creates security risks and may result in fund loss.

### Auto-Signing
- Automatic transaction signing without user approval
- Silent signing mechanisms
- Bypassing wallet confirmation dialogs
- Auto-approval of transactions

### Silent Approvals
- Approvals without user awareness
- Hidden token allowances
- Approvals without risk disclosure
- Approvals without explicit user confirmation

### Bots or Schedulers
- Automated trading systems
- Scheduled transaction execution
- Event-triggered automatic transactions
- Unattended execution logic
- Time-based automatic operations

### Background Execution
- Transactions executed without user presence
- Background transaction submission
- Silent transaction processing
- Unattended transaction execution

### Key Handling
- Private key generation
- Mnemonic generation
- Key storage or management
- Key recovery mechanisms
- Any custodial key handling

### MEV Logic
- Front-running implementations
- Sandwich attack strategies
- MEV extraction mechanisms
- Transaction ordering manipulation
- Arbitrage automation

### Gas-Blind Execution
- Transactions without gas estimation
- Transactions without fee disclosure
- Gas optimization without user awareness
- Transactions without cost visibility
- Execution that hides gas costs

## Summary

These patterns are mandatory for safe Avalanche C-Chain development. Security always overrides convenience. When in doubt, default to read-only operations and require explicit user confirmation.

**Key Principles:**
- Read-only by default
- Explicit user confirmation for all writes
- simulate-before-send is mandatory
- Chain validation is required
- Client separation is non-negotiable
- Security overrides convenience

If a pattern is not defined here, it is likely forbidden. When uncertain, choose the most restrictive approach that maintains security.
