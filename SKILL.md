---
name: avalanche-dev
description: End-to-end Avalanche C-Chain development playbook (EVM-first, Foundry-only)
user-invocable: true
---

# avalanche-dev

## What this Skill is for

This skill provides end-to-end development capabilities for Avalanche C-Chain applications. It is an opinionated, security-first playbook for building production EVM contracts and dApps on Avalanche.

**Scope:**
- Smart contract development, testing, and deployment on Avalanche C-Chain
- Frontend integration with Avalanche wallets and RPC providers
- Security-first development practices and validation
- Production deployment workflows

**Out of scope:**
- Other Avalanche subnets (P-Chain, X-Chain)
- Non-EVM chains
- Educational tutorials or beginner guides

## Default Stack Decisions

### Chain Configuration
- **Primary Chain**: Avalanche C-Chain Mainnet (Chain ID: 43114)
- **Testnet**: Fuji Testnet (Chain ID: 43113) for pre-production testing
- **Local Development**: anvil (Foundry) with mainnet fork capability
- **No other chains permitted** in default workflows

### Client Library
- **Primary**: viem (latest stable)
  - Use `WalletClient` for signing operations
  - Use `PublicClient` for read operations
  - Prefer viem's type system and composability
- **Legacy Boundary**: ethers.js v6+ (only when interfacing with legacy systems)
  - Must be explicitly justified
  - Prefer migration to viem
- **Forbidden**: web3.js (explicitly discouraged, use only as last resort with explicit warning)

### Client Boundary Rules (Non-Negotiable)
- Read-only blockchain operations MUST use viem `PublicClient`
- Signing and transaction submission MUST use `WalletClient`
- Never reuse a signing client for read-only operations
- Never mix read/write responsibilities in the same client instance
- Maintain strict separation between public and wallet client instances

### Wallet Integration
- **First-Class**: Core Wallet (Avalanche native)
- **Standard**: WalletConnect v2
- **Browser Injection**: Explicitly discouraged unsafe patterns
  - Never auto-approve transactions
  - Require explicit user confirmation for all signing operations
  - Validate chain ID before any transaction
  - Display fee estimates before signing
- Wallet private keys MUST NEVER be logged, persisted, serialized, or exposed to skills, logs, or sub-agents.

### Smart Contract Language
- **Language**: Solidity >= 0.8.x (latest stable)
- **Libraries**: OpenZeppelin Contracts (audited, battle-tested)
- **Upgradeability**: Disabled by default
  - Only enable if explicitly requested
  - Must use transparent proxy pattern with explicit security review
  - Document upgrade risks and procedures

### RPC Providers
- **Primary**: Avalanche Public RPC endpoints
- **Production**: Infura, Alchemy, or QuickNode (with rate limiting and monitoring)
- **Local**: anvil (Foundry) for development and testing

## Operating Procedure

### 1. Project Initialization
- Initialize Foundry project structure (`forge init`)
- Configure `foundry.toml` with Avalanche network settings
- Set up environment variables for private keys and RPC URLs
- Configure `.env` with separate keys for testnet and mainnet
- Install viem and required dependencies

### 2. Smart Contract Development
- Write Solidity contracts following OpenZeppelin patterns
- Use Solidity 0.8.x+ built-in overflow protection (no SafeMath needed)
- Implement access control using OpenZeppelin's `Ownable` or `AccessControl`
- Follow patterns defined in `patterns.md`
- Use events for all state-changing operations (off-chain indexing)
- Optimize for Avalanche's gas structure (lower fees than Ethereum mainnet)

### 3. Testing Strategy
- Write all tests in Solidity using Foundry
- Use `forge test` for running test suite
- Use `anvil` for local development and mainnet fork testing
- Test on local anvil instance first
- Fork Fuji Testnet for integration testing (`anvil --fork-url <fuji-rpc>`)
- Fork Mainnet for final validation before deployment
- Use `cast` for contract inspection and debugging
- Achieve 100% branch coverage before mainnet deployment
- Test all edge cases, error conditions, and access control paths

### 4. Security Review
- Run Slither static analysis on all contracts
- Review against `security.md` checklist
- Perform manual security review focusing on:
  - Reentrancy vulnerabilities
  - Access control bypasses
  - Integer overflow/underflow (though protected in 0.8.x)
  - Front-running vectors
  - Gas griefing attacks
- Fork test all critical paths on mainnet fork
- Validate all external calls and library dependencies

### 5. Deployment Process
- Deploy to Fuji Testnet first using `forge script`
- Verify contracts on Snowtrace (Avalanche block explorer)
- Test all functionality end-to-end on testnet
- Validate gas costs and transaction patterns
- Deploy to Mainnet with explicit confirmation
- Verify contracts on Snowtrace Mainnet
- Update frontend with verified contract addresses and ABIs
- Document all contract addresses, ABIs, and deployment parameters

### 6. Frontend Integration
- Use viem's `createWalletClient` and `createPublicClient` with strict separation
- Configure clients with Avalanche C-Chain network parameters
- Implement explicit chain validation before any transaction
- Use `simulateContract` before `writeContract` (simulate-before-send)
- Display transaction details and fee estimates to user
- Require explicit user confirmation for all signing operations
- Wallet support: Core Wallet (first-class), WalletConnect v2 (standard)
- MetaMask allowed ONLY via explicit user choice or WalletConnect v2 connection
- Do NOT implicitly endorse unsafe browser injection patterns
- Never auto-approve or silently sign transactions
- Chain switching must be user-initiated, never automatic
- Implement proper error handling and user feedback

### 7. Monitoring and Maintenance
- Monitor contract interactions via Snowtrace
- Set up alerts for critical events and anomalies
- Maintain upgradeable contracts with documented upgrade procedures
- Document all contract addresses, ABIs, and deployment configurations
- Track gas usage patterns and optimize as needed

## Testing Strategy

### Why Foundry (Non-Negotiable)

Foundry is the only acceptable testing framework for this skill. This decision is non-negotiable for the following reasons:

1. **Native Solidity Testing**: Tests are written in Solidity, eliminating context switching and enabling direct contract interaction patterns
2. **Performance**: Foundry's Rust-based execution is orders of magnitude faster than JavaScript-based test runners
3. **Fork Testing**: Built-in support for mainnet/testnet forking via anvil enables realistic testing environments
4. **Gas Profiling**: Native gas reporting and optimization tools built into the framework
5. **Type Safety**: Solidity tests provide compile-time type checking and better IDE integration
6. **Debugging**: Superior debugging capabilities with `forge test --debug` and stack traces
7. **Industry Standard**: Foundry is the de facto standard for EVM development in production environments

**Forbidden Testing Frameworks:**
- Hardhat (JavaScript-based, slower, unnecessary abstraction)
- Truffle (legacy, deprecated patterns)
- Waffle (redundant with Foundry)
- Any npm-based test runners (Jest, Mocha, etc.)

### Foundry Testing Workflow

1. **Local Testing**: Run `forge test` against anvil local node
2. **Fork Testing**: Use `anvil --fork-url` to test against Fuji or Mainnet state
3. **Gas Reporting**: Use `forge test --gas-report` for optimization analysis
4. **Coverage**: Use `forge coverage` to ensure comprehensive test coverage
5. **Fuzz Testing**: Leverage Foundry's built-in fuzzing capabilities
6. **Invariant Testing**: Use Foundry's invariant testing for complex protocols

### Test Structure Requirements
- All tests must be in Solidity
- Use Foundry's `Test` contract as base
- Organize tests by contract functionality
- Include both unit tests and integration tests
- Test access control, error conditions, and edge cases
- Use fixtures for complex setup scenarios

## Security Checklist

Before any mainnet deployment, verify:

- [ ] All contracts pass Slither static analysis with no high/critical issues
- [ ] Reentrancy guards implemented for all external calls
- [ ] Access control verified for all privileged functions
- [ ] Integer overflow/underflow protected (Solidity 0.8.x+)
- [ ] Front-running vectors identified and mitigated
- [ ] All external calls validated and sanitized
- [ ] Events emitted for all state-changing operations
- [ ] Fork testing completed on mainnet fork
- [ ] Gas optimization reviewed without compromising security
- [ ] Upgrade procedures documented (if applicable)
- [ ] Emergency pause mechanisms tested (if applicable)
- [ ] Review against `security.md` completed

**Never deploy without:**
- Complete test coverage (100% for critical paths)
- Security review sign-off
- Fork testing validation
- Explicit user confirmation

## Deliverables Expectations

### Smart Contracts
- Solidity source code with comprehensive NatSpec documentation
- Compiled artifacts (ABI, bytecode)
- Deployment scripts (Foundry scripts)
- Verification on Snowtrace

### Testing
- Complete Foundry test suite with 100% coverage
- Fork test results (Fuji and Mainnet)
- Gas optimization reports
- Security analysis reports (Slither)

### Documentation
- Contract interface documentation
- Deployment addresses and network configurations
- ABI files for frontend integration
- Security audit findings (if applicable)

### Frontend Integration
- viem client configuration
- Wallet integration (Core Wallet, WalletConnect)
- Transaction simulation and signing flows
- Error handling and user feedback
- Clear wallet-risk disclosure for end users

## Progressive Disclosure / Reference Docs

### Primary References
- `patterns.md`: Avalanche-specific development patterns and best practices
- `security.md`: Comprehensive security guidelines, vulnerability patterns, and audit checklist
- `rules.md`: Development rules, constraints, and forbidden patterns
- `IDENTITY.md`: Skill identity and behavioral guidelines

### External References
- [Avalanche C-Chain Documentation](https://docs.avax.network/learn/avalanche)
- [Foundry Book](https://book.getfoundry.sh)
- [viem Documentation](https://viem.sh)
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts)
- [Snowtrace Explorer](https://snowtrace.io)
- [Solidity Documentation](https://docs.soliditylang.org)

### Network Endpoints
- **Mainnet RPC**: `https://api.avax.network/ext/bc/C/rpc`
- **Fuji Testnet RPC**: `https://api.avax-test.network/ext/bc/C/rpc`
- **Mainnet Explorer**: `https://snowtrace.io`
- **Fuji Explorer**: `https://testnet.snowtrace.io`
- **Fuji Faucet**: `https://faucet.avalanche.org`

## Security Posture

This skill enforces a strict security-first approach:

1. **Simulate-Before-Send**: Always use `simulateContract` before `writeContract` to validate transaction outcomes
2. **No Silent Signing**: All transaction signing requires explicit user confirmation with visible transaction details
3. **Explicit Fee Visibility**: Display gas estimates and total costs before user approval
4. **Signer Validation**: Verify signer address and chain ID before any operation
5. **Replay Protection**: Ensure all transactions use proper nonce management and chain-specific replay protection
6. **Fork Testing**: Mandatory mainnet fork testing before any production deployment
7. **Access Control**: Explicit access control checks for all privileged operations
8. **Event Logging**: All state changes must emit events for off-chain monitoring
9. **Key Protection**: Skills MUST NOT request, store, or transmit private keys, seed phrases, or raw signatures unless explicitly user-approved for a single transaction

Refer to `security.md` for comprehensive security requirements and detailed audit procedures.

## Explicit Non-Goals

This skill will NOT provide:
- Wallet custody or key management services
- Automated trading or transaction execution without explicit user approval
- MEV (Maximum Extractable Value) strategies or front-running mechanisms
- Private key generation, storage, or recovery service
