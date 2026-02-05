# avalanche-dev

A security-first development skill for building production-grade applications on Avalanche C-Chain.

## What is avalanche-dev?

`avalanche-dev` is an opinionated, security-first development skill designed for Avalanche C-Chain (EVM) development. It focuses on reducing irreversible mistakes on mainnet while maintaining production-grade development practices.

**Core Design Principles:**
- **Security-first**: Every decision prioritizes safety over convenience
- **Production-minded**: Built for real deployments, not just learning
- **Opinionated by design**: Enforces best practices to reduce risk
- **Explicit confirmation**: Never assumes intent for irreversible actions

**Technical Stack:**
- **Chain**: Avalanche C-Chain only (Mainnet: 43114, Fuji Testnet: 43113)
- **Testing**: Foundry only (forge, anvil, cast)
- **Client**: viem-first (ethers.js only as legacy boundary)
- **Smart Contracts**: Solidity ≥0.8.x with OpenZeppelin patterns

This skill is designed for developers who understand EVM fundamentals and need a security-focused partner for Avalanche C-Chain development.

## When should you use this skill?

Use `avalanche-dev` when you need to:

- **Write or review Solidity contracts** for Avalanche C-Chain deployment
- **Build frontend integrations** with Core Wallet, WalletConnect v2, or MetaMask
- **Prepare contracts for mainnet deployment** with proper testing and validation
- **Review transaction safety** including approvals, gas behavior, and execution risks
- **Simulate and validate transactions** before execution to prevent errors
- **Set up Foundry testing** for Avalanche-specific contract patterns
- **Validate wallet integration patterns** for security and correctness

**Default Behavior:**
- The skill defaults to **read-only analysis** and explanation
- It escalates to execution only after **explicit user confirmation**
- It requires clear understanding of risks before any state-changing operation

## What this skill will NOT do (Important)

These are intentional safety boundaries, not missing features:

- **Will not manage or store private keys**: Wallets are external trust boundaries
- **Will not auto-sign or auto-execute transactions**: Every transaction requires explicit user approval
- **Will not build trading bots or automation**: Unattended execution creates security risks
- **Will not execute MEV, arbitrage, or sandwich strategies**: These patterns harm users and violate safety principles
- **Will not bypass wallet confirmations**: All signing requires user action via wallet software
- **Will not operate on non–C-Chain Avalanche networks**: P-Chain and X-Chain are out of scope
- **Will not deploy to mainnet without testnet validation**: Fuji testnet testing is mandatory
- **Will not provide custody or key recovery**: Key management is outside the skill's scope

These boundaries exist to protect users from irreversible mistakes and fund loss. The skill will refuse requests that violate these boundaries, even if the user insists.

## How avalanche-dev works (High-level)

The skill operates on a security-first model:

### Read-Only by Default
All blockchain interactions default to read-only analysis. The skill assumes you want to understand, simulate, or review before executing.

### Explicit Confirmation for Writes
State-changing operations require explicit user confirmation. The skill explains what will happen, shows risks, and waits for your approval before proceeding.

### Simulate-Before-Send Enforced
Every transaction is simulated before submission. If simulation fails, the transaction is not sent. You see the simulation results before signing.

### Wallets as External Trust Boundaries
Wallets (Core Wallet, WalletConnect, MetaMask) are treated as external systems. The skill never requests private keys or bypasses wallet security. All signing happens through wallet software with your explicit approval.

### Mainnet as Irreversible and Final
Mainnet operations are treated as final and irreversible. The skill requires explicit confirmation, testnet validation, and clear understanding of risks before any mainnet action.

## Supported Stack (Summary)

| Component | Technology |
|-----------|-----------|
| **Chain** | Avalanche C-Chain (Mainnet: 43114, Fuji: 43113) |
| **Testnet** | Fuji Testnet (43113) |
| **Smart Contracts** | Solidity ≥0.8.x |
| **Testing** | Foundry (forge, anvil, cast) |
| **Frontend Client** | viem (latest stable) |
| **Wallets** | Core Wallet (first-class), WalletConnect v2 (standard) |
| **Block Explorer** | Snowtrace (Mainnet & Fuji) |
| **Libraries** | OpenZeppelin Contracts |

## Typical Usage Examples (Conceptual)

The skill is designed for interactive development workflows:

**Contract Development:**
- "Review this Solidity contract for Avalanche mainnet safety"
- "Help me write a Foundry test for this contract function"
- "Validate that this access control pattern is secure"

**Transaction Safety:**
- "Simulate this transaction before I send it"
- "Explain what this approval will allow"
- "Check if this transaction will succeed on mainnet"

**Frontend Integration:**
- "Validate that this wallet connection flow is safe"
- "Review this viem client configuration for Avalanche"
- "Check if this transaction flow follows security best practices"

**Deployment Preparation:**
- "Help me prepare this contract for mainnet deployment"
- "Review my Foundry deployment script"
- "Validate my contract addresses before deployment"

All examples are conceptual. The skill provides guidance, validation, and safety checks, but you control execution through your wallet.

## Safety & Responsibility Notice

**How the skill helps:**
- Provides security-focused guidance and validation
- Enforces best practices and safety patterns
- Requires explicit confirmation for risky operations
- Simulates transactions before execution
- Validates chain IDs and contract addresses

**Your responsibility:**
- You always control your wallet and private keys
- You approve all transactions through wallet software
- You are responsible for verifying transaction details
- You are responsible for understanding what you sign
- Final responsibility for signed transactions lies with you

**Mainnet reality:**
- Mainnet actions are irreversible
- Real funds are at risk on mainnet
- Mistakes can result in permanent fund loss
- Always test on Fuji testnet first
- Always validate before mainnet deployment

The skill reduces risk but cannot eliminate it. You remain responsible for your wallet and transactions.

## Extending the Skill Safely

If you need to extend or customize the skill:

**Follow the patterns:**
- New patterns should align with `patterns.md`
- Use approved viem client separation patterns
- Maintain simulate-before-send for all transactions
- Validate chain IDs before any operation

**Respect the rules:**
- All extensions must respect `rules.md` constraints
- Security rules override convenience
- Unsafe extensions should be refused, not implemented
- Never bypass security boundaries

**Security first:**
- `security.md` always overrides convenience
- When in doubt, choose the most restrictive approach
- Default to read-only when uncertain
- Require explicit confirmation for risky operations

**Refusal is safety:**
- The skill will refuse unsafe extensions
- Refusal protects users from irreversible mistakes
- Do not attempt to bypass security boundaries
- If the skill refuses, there is a security reason

## Related Files

This skill is defined by several documents:

- **SKILL.md**: Core skill definition, scope, stack decisions, and operating procedures
- **rules.md**: Non-negotiable hard rules that override user requests
- **security.md**: Threat model, security risks, and mandatory mitigations
- **patterns.md**: Approved, security-reviewed implementation patterns
- **IDENTITY.md**: Agent persona, behavior, tone, and refusal style

These documents work together to ensure safe, production-grade Avalanche C-Chain development. When in doubt, refer to these documents for authoritative guidance.

---

**Remember**: Security always overrides convenience. When uncertain, default to read-only and require explicit confirmation.
