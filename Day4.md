## Day4

## Protocol-Level Security Considerations
1. **Implementation of Secure Delegate Contracts**
    - Replay Protection: Delegate contracts must enforce nonces in signatures to prevent reuse of authorization tuples; without this, an attacker can replay a valid signature to repeat privileged actions.

    - Signed Fields: Critical parameters such as value, gas, target, and calldata must be signed; omitting any allows a malicious sponsor to manipulate funds, grief by draining gas, or redirect calls arbitrarily.

2. **Front-Running Initialization**
    - Lack of Initcode: Since constructors don’t run during delegation, storage slots remain uninitialized, enabling observers to front-run the intended setup by delegating an account they control if initialization isn’t properly guarded.
    - Signed Initialization: Wallets must require the initial calldata (e.g. initialize calls) to be signed by the EOA’s key via ecrecover, ensuring only the legitimate owner can set up critical state.

3. **Storage Management**
    - Persistent State Across Delegations: Changing the delegate contract does not clear existing storage. If a new contract’s layout isn’t carefully re-rooted (e.g. via unique storage slots per implementation), data collisions can cause undefined behavior or compromise account invariants.

4. **Breaking tx.origin == msg.sender Invariants**
    - Self-Sponsoring: Allowing tx.origin to also set code breaks assumptions in contracts that rely on require(tx.origin == msg.sender) for atomic sandwich protection or reentrancy guards, potentially nullifying those protections.

5. **Sponsored Transaction Relayer Abuse**
    - Gas Theft: An attacker can craft authorizations that cause relayers to spend gas without reimbursement—either by invalidating authorizations via nonce bumps or by sweeping assets—if relayers lack bonds or reputational checks.

6. **Transaction Propagation Challenges**
    - Stale Transactions: Once an EOA delegates, its balance and nonce can change unpredictably during another transaction, making it impossible for nodes to statically validate pending transactions; clients should limit to one pending transaction per delegated EOA or extend the EIP with hydration lists.

## Implementation-Level Attack Surfaces
1. **Access Control Failures**
    - Unrestricted Functions: Delegate contracts without proper onlyOwner-style guards (e.g., functions like doSomething that call arbitrary addresses) allow any user to execute transactions on the EOA’s behalf, leading to unauthorized fund transfers.

2. **Initialization Challenges**
    - Constructor Blindspot: Delegation bypasses constructors, so any default values set there are never applied. Naïve assumptions about initial state can be exploited by attackers who provide malicious delegate code with unexpected defaults.
    - Re-Initialization Front-Running: If delegate contracts use an initializer pattern without strict access checks, attackers can front-run the legitimate user’s initialize call—running it themselves first and locking out the real owner.

3. **Storage Collisions**
    - Layout Mismatch: Migrating from one delegate contract (e.g., ContractA with a bool) to another (ContractB with a uint) leads to misinterpreted storage slots, causing logic errors or enabling state-corruption exploits.