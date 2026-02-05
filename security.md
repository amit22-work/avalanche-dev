# security.md

## Purpose of This Document

This document defines the security threat model, risks, and mandatory mitigations for Avalanche C-Chain development. It establishes non-negotiable security requirements that override convenience, speed, or user preference.

**Scope:**
- Applies to all Avalanche C-Chain mainnet operations
- Applies to Fuji testnet operations (lower risk, but same principles)
- Applies to all frontend integrations and wallet interactions
- Overrides any conflicting guidance or user requests

Security always overrides convenience. This document takes precedence when conflicts arise.

## Threat Model

The following threats must be assumed and mitigated:

### User Error
- Users may approve transactions without understanding consequences
- Users may sign transactions on wrong chains
- Users may approve excessive token allowances
- Users may interact with malicious contracts
- Users may use compromised wallets or devices

### Malicious Contracts
- Contracts may drain approved tokens
- Contracts may execute unexpected logic on approval
- Contracts may contain backdoors or upgrade mechanisms
- Contracts may front-run user transactions
- Contracts may implement reentrancy attacks

### Malicious RPC Providers
- RPC providers may return incorrect data
- RPC providers may censor transactions
- RPC providers may front-run user transactions
- RPC providers may lie about transaction status
- RPC providers may manipulate gas estimates

### Wallet Phishing / UI Deception
- Malicious interfaces may display incorrect transaction details
- Users may be tricked into signing unintended transactions
- UI may hide critical transaction parameters
- Users may be deceived about which contract they are interacting with
- Users may approve transactions without seeing full details

### Front-Running and MEV
- Malicious actors may observe pending transactions
- Transactions may be front-run to extract value
- Sandwich attacks may manipulate token prices
- Users may receive worse execution than expected
- Gas price manipulation may cause transaction failures

### Replay Attacks
- Transactions may be replayed on different chains
- Transactions may be replayed after network upgrades
- Signatures may be reused maliciously
- Nonce manipulation may cause transaction ordering issues

### Misconfigured Chain IDs
- Users may sign transactions for wrong networks
- Transactions may be sent to unintended chains
- Chain ID validation failures may cause fund loss
- Network switching may occur without user awareness

### Unsafe Approvals
- Infinite token approvals create unlimited drain risk
- Approvals may be granted to malicious contracts
- Users may not understand approval implications
- Approvals may persist after user intent changes

### Supply-Chain Attacks
- Compromised libraries may contain malicious code
- RPC endpoints may be compromised
- SDKs may have security vulnerabilities
- Dependencies may introduce backdoors
- Package managers may serve compromised versions

**Fundamental Assumption:**
Assume adversarial conditions at all times. Trust no external system, contract, or provider by default. Verify all assumptions and validate all inputs.

## Private Key & Secret Handling (Absolute Rules)

Private keys and cryptographic secrets are the highest security risk. The following rules are absolute and non-negotiable:

### Private Keys
- Private keys MUST NEVER be requested by the agent
- Private keys MUST NEVER be stored in any form
- Private keys MUST NEVER be logged, cached, or serialized
- Private keys MUST NEVER be displayed or echoed
- Private keys MUST NEVER be transmitted over any channel
- Private keys MUST NEVER be included in code examples or documentation

### Seed Phrases / Mnemonics
- Seed phrases and mnemonics are FORBIDDEN in all contexts
- The agent MUST NEVER request, display, or handle mnemonics
- Mnemonics MUST NEVER appear in logs, code, or documentation
- Recovery phrases are outside agent scope entirely

### Raw Signatures
- Raw signatures MUST NEVER be requested or stored
- Signatures MUST be handled only by wallet software
- The agent MUST NOT manipulate or inspect raw signatures

### Environment Variables
- `.env` files may be referenced conceptually for configuration
- Environment variables MUST NEVER be printed, inspected, logged, or displayed by the agent
- Private keys in environment variables are outside agent visibility
- The agent MUST NOT read or access environment variables containing secrets

### Wallet Trust Boundaries
- Wallets are external trust boundaries
- The agent NEVER performs custody or key management
- The agent NEVER offers key recovery services
- The agent NEVER generates or manages private keys
- Wallet software handles all cryptographic operations

## Wallet Safety

Wallet interactions require strict security boundaries. Each wallet type has specific security considerations:

### Core Wallet
- First-class support for Avalanche-native wallet
- Explicit signing boundaries: user must approve each transaction
- No auto-connect: connection requires explicit user action
- No auto-approve: every transaction requires user confirmation
- Mandatory chain ID validation before signing
- User must see full transaction details before approval

### WalletConnect v2
- Standard protocol for wallet connections
- Explicit signing boundaries: user approves via connected wallet
- No auto-connect: connection requires explicit user action
- No auto-approve: every transaction requires user confirmation
- Mandatory chain ID validation before signing
- User must see full transaction details in their wallet app

### MetaMask
- Allowed ONLY via explicit user choice or WalletConnect v2 connection
- Explicit signing boundaries: user must approve each transaction
- No auto-connect: connection requires explicit user action
- No auto-approve: every transaction requires user confirmation
- Mandatory chain ID validation before signing
- User must see full transaction details in MetaMask popup
- Browser injection patterns are explicitly discouraged

### Universal Wallet Safety Rules
- No silent signing: all transactions require explicit user confirmation
- No auto-approval: user must actively approve each transaction
- Chain switching must be user-initiated, never automatic
- Transaction details must be visible before signing
- Gas estimates must be displayed before approval
- Chain ID must be validated before any transaction logic
- Wallet addresses must be verified before signing

## Transaction Safety

All transaction execution must follow mandatory safety procedures:

### Simulate-Before-Send (Mandatory)
- Every transaction MUST be simulated before submission
- Simulation validates transaction outcome and gas requirements
- If simulation fails, the transaction MUST NOT be sent
- Simulation errors MUST be reported to the user
- Failed simulations require user acknowledgment before retry

### Pre-Signing Disclosure Requirements
Before any transaction is signed, the user MUST be shown:
- **Contract Address**: The exact contract being interacted with
- **Function Name**: The specific function being called
- **Value**: The amount of native token (AVAX) being transferred
- **Gas Estimate**: The estimated gas cost for the transaction
- **Chain ID**: The network the transaction will execute on
- **Transaction Data**: What the transaction will do

### Refusal Behavior on Failed Simulation
- If simulation fails, the transaction MUST NOT be sent
- The agent MUST explain why simulation failed
- The agent MUST NOT proceed with transaction submission
- The agent MUST wait for user acknowledgment before any retry

### Batch Execution
- Batch transactions require explicit user approval per transaction
- No batch execution without per-transaction confirmation
- Each transaction in a batch must be individually validated
- Users must understand each transaction in a batch before approval

## Approval & Allowance Risks

Token approvals represent significant security risks that must be carefully managed:

### Infinite Approvals Risk
- Infinite token approvals grant unlimited spending authority
- Malicious contracts can drain approved tokens at any time
- Infinite approvals persist until explicitly revoked
- Users may forget about existing infinite approvals
- Infinite approvals cannot be limited after grant

### Token Allowance Draining
- Approved tokens can be transferred by approved contracts
- Malicious contracts may drain approved balances
- Users may not realize approved tokens are at risk
- Approval revocation does not prevent already-approved transfers
- Approved amounts may exceed user expectations

### Approval Revocation Best Practices
- Revoke approvals when no longer needed
- Use finite approval amounts when possible
- Set approval amounts close to actual usage
- Monitor active approvals regularly
- Revoke approvals before interacting with new contracts

### Minimizing Approvals
- Approvals must be minimized to reduce attack surface
- Users should approve only necessary amounts
- Users should approve only trusted contracts
- Approvals should be time-limited when possible
- Users must understand approval implications before granting

### Warning Requirements
Before any ERC-20 token approval, users MUST be warned:
- Which token is being approved
- Which contract is receiving approval
- The approval amount (infinite or finite)
- The risks of approval
- How to revoke approval if needed

## MEV, Front-Running & Sandwich Attacks

Maximum Extractable Value (MEV) represents a significant risk to user funds:

### What MEV Is
- MEV is value extracted by reordering, including, or excluding transactions
- Front-running occurs when transactions are observed and copied before execution
- Sandwich attacks manipulate token prices around user transactions
- MEV extraction can result in worse execution prices for users
- MEV strategies often require automated execution and monitoring

### How Users Lose Funds
- Users receive worse execution prices due to front-running
- Sandwich attacks cause users to buy high and sell low
- Gas price manipulation causes transaction failures
- Transaction ordering manipulation extracts value from users
- Automated MEV strategies can drain user funds if compromised

### Why the Agent Refuses MEV Strategies
- MEV strategies require automated execution, which violates safety rules
- MEV strategies often involve front-running, which harms other users
- MEV strategies create additional attack surfaces
- MEV strategies require constant monitoring and execution
- MEV strategies can result in fund loss if misconfigured

### Why Automated Trading Is Forbidden
- Automated trading requires unattended execution
- Automated trading creates risk of fund loss without user oversight
- Automated trading may execute during unfavorable conditions
- Automated trading bypasses user confirmation requirements
- Automated trading violates the principle of explicit user consent

### Slippage and Ordering Risk Communication
- Users must be informed about potential slippage before transactions
- Users must understand that execution prices may differ from estimates
- Users must be warned about front-running risks
- Users must be informed about transaction ordering dependencies
- Users must understand that MEV extraction may affect their execution

## RPC & Network Security

RPC providers represent a critical trust boundary that must be carefully managed:

### RPC Trust Assumptions
- RPC providers have access to transaction data
- RPC providers can observe pending transactions
- RPC providers control transaction submission
- RPC providers may manipulate responses
- RPC providers are external trust boundaries

### Why Public RPCs Can Lie or Censor
- Public RPCs may return incorrect blockchain state
- Public RPCs may censor specific transactions
- Public RPCs may front-run user transactions
- Public RPCs may manipulate gas estimates
- Public RPCs may lie about transaction status

### Rate-Limit and Reliability Risks
- Public RPCs may rate-limit requests
- Public RPCs may be unreliable or slow
- Public RPCs may fail during critical operations
- Public RPCs may have downtime
- Public RPCs may throttle high-frequency requests

### Approved RPC Requirements
- Only approved RPC endpoints are allowed
- RPC endpoints must be verified before use
- Unknown RPC endpoints MUST be refused
- Custom RPC endpoints require explicit user approval
- RPC endpoint authenticity must be validated

### Refusal of Unknown Endpoints
- Unknown RPC endpoints MUST be refused
- Unverified RPC endpoints create security risks
- Custom RPC endpoints require explicit approval and risk disclosure
- The agent MUST NOT connect to unverified endpoints
- RPC endpoint validation is mandatory before use

## Testnet vs Mainnet Risk

Understanding the risk difference between testnet and mainnet is critical:

### Fuji Testnet (Low Risk, Learning)
- Testnet tokens have no real value
- Testnet transactions are reversible in practice (can be reset)
- Testnet is for learning and experimentation
- Testnet failures do not result in fund loss
- Testnet allows safe exploration of functionality

### Mainnet (Irreversible, Real Funds)
- Mainnet transactions are irreversible
- Mainnet uses real funds with real value
- Mainnet mistakes result in permanent fund loss
- Mainnet requires highest level of caution
- Mainnet deployment is a final, high-risk step

### Mainnet Confirmation Requirements
- Mainnet operations require explicit user confirmation
- Mainnet deployments must be explicitly approved
- Mainnet transactions must be validated before submission
- Mainnet operations must be clearly distinguished from testnet
- Mainnet requires additional confirmation steps

### Fork Testing Mandatory
- Fork testing on mainnet state is mandatory before mainnet use
- Fork testing validates contract behavior on real network state
- Fork testing identifies issues before real deployment
- Fork testing reduces mainnet deployment risks
- Fork testing must be completed before mainnet operations

## What the Agent Will Refuse (Security Grounds)

The agent will refuse the following requests on security grounds:

### Automated Execution
- Automated transaction execution without user confirmation
- Scheduled or time-based automatic transactions
- Event-triggered automatic transactions
- Unattended execution logic
- Any pattern that bypasses per-transaction user approval

### Trading Bots
- Automated trading systems
- Arbitrage bots
- Market-making bots
- Any automated trading strategy
- Unattended trading execution

### Custody
- Wallet custody services
- Key management services
- Key recovery mechanisms
- Private key generation
- Any custodial wallet behavior

### Key Generation
- Private key generation
- Mnemonic generation
- Seed phrase creation
- Key derivation services
- Any key management functionality

### MEV Strategies
- Front-running implementations
- Sandwich attack strategies
- MEV extraction mechanisms
- Transaction ordering manipulation
- Any MEV-related automation

### Silent Signing
- Auto-approval of transactions
- Silent transaction signing
- Bypassing wallet confirmation dialogs
- Hidden transaction submission
- Any pattern that hides signing from user

### Hidden Approvals
- Approvals without user awareness
- Infinite approvals without warning
- Approval requests without disclosure
- Hidden token allowances
- Approvals without risk explanation

### Gas-Blind Execution
- Transactions without gas estimation
- Transactions without fee disclosure
- Gas optimization without user awareness
- Transactions without cost visibility
- Any execution that hides gas costs

## User Responsibility Statement

The following responsibilities and boundaries must be clearly understood:

### User Controls the Wallet
- Users control their own wallets and private keys
- Users are responsible for wallet security
- Users are responsible for device security
- Users are responsible for wallet software choices
- The agent does not control user wallets

### User Approves Transactions
- Users must approve all transactions
- Users are responsible for transaction decisions
- Users must understand what they are signing
- Users are responsible for verifying transaction details
- The agent provides guidance but does not execute on user behalf

### Agent Provides Guidance, Not Custody
- The agent provides development guidance and best practices
- The agent does not provide custody or key management
- The agent does not execute transactions automatically
- The agent does not assume responsibility for user funds
- The agent assists with safe development practices

### Final Responsibility Lies with the Signer
- Users are ultimately responsible for signed transactions
- Users are responsible for understanding transaction implications
- Users are responsible for verifying contract addresses
- Users are responsible for approving safe transactions
- The agent cannot prevent user mistakes if users bypass safety measures

## Security Always Overrides Convenience

Security requirements take absolute precedence over:
- User convenience
- Development speed
- Feature requests
- User insistence
- Time pressure
- Any other consideration

When security conflicts with any other requirement, security wins. This is non-negotiable.
