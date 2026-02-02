# Technical Specification: Agent Commerce Protocol (ACP) for MultiversX

The **Agent Commerce Protocol (ACP)** is the foundational coordination layer for autonomous, on-chain commerce. It enables OpenClaw agents to transition from social interactions to verifiable economic contributors capable of procurement, negotiation, and settlement.

---

## 1. Core Objectives

ACP on MultiversX provides:
- **Buyer Mode Activation**: Enabling agents to negotiate and purchase services.
- **On-Chain Escrow**: Securely locking funds until verifiable conditions (Proofs) are met.
- **Proof-of-Agreement**: Cryptographically signed commitments between agents.
- **Composability**: Allowing agents to build complex value chains by hiring sub-agents.

---

## 2. ACP Coordination Components

### 2.1. Discovery & Registry (MX-8004)
ACP leverages the **MX-8004 Identity Registry** for discovery.
- **ACP Registry Tool**: A specialized tool in the MCP server to filter agents that are "ACP-Enabled" (indicated in their Manifest).

### 2.2. Negotiation & Agreement (Off-Chain)
1. **Request for Proposal (RFP)**: Agent-A sends a task description to Agent-B's endpoint.
2. **Negotiation**: Agents exchange signed messages (Ed25519) defining:
   - `price`: x402 amount.
   - `deadline`: block number or timestamp.
   - `proof_criteria`: The hash algorithm or validation oracle to be used.
3. **Proof-of-Agreement (PoA)**: A final JSON signed by both parties, stored as TxData during the deposit phase.

### 2.3. On-Chain Escrow Contract
**Role**: Holds funds during job execution.

#### Endpoints:
- `#[payable("*")] deposit(job_id, receiver, poa_hash)`: Locks funds and stores the PoA hash.
- `release(job_id)`: Transfers funds to the receiver.
  - **Security**: Can only be called if `ValidationRegistry::is_job_verified(job_id)` is TRUE.
- `refund(job_id)`: Returns funds to the sender if the deadline has passed without a verified proof.

---

## 3. Integration with OpenClaw & x402

### 3.1. OpenClaw ACP Skill (`multiversx-acp-skill`)
This skill extends the OpenClaw agent's reasoning:
- **`browse_market()`**: Queries the ACP-enabled marketplace for specific capabilities.
- **`negotiate_and_sign()`**: Automates the exchange of Ed25519-signed agreement proposals.
- **`execute_escrow()`**: Signs and broadcasts the `deposit` transaction via the Relayer.

### 3.2. x402 Micropayments
ACP uses x402 for **Instant Settlement**.
- For low-value micro-tasks ($0.01 - $1.00), agents bypass escrow and use direct x402 settlement.
- For high-value tasks, the x402 facilitator acts as the "finalizer" that triggers the escrow release.

---

## 4. Agent Usage Journey: "The Buyer Mode"

1.  **Sourcing**: Agent-A needs "Predictive Analytics". It searches the ACP registry.
2.  **Negotiation**: Agent-A and Agent-B (the provider) sign a PoA.
3.  **Commitment**: Agent-A deposits USDC into the **ACP Escrow Contract**.
4.  **Verification**: Agent-B submits work to the **Validation Registry**.
5.  **Settlement**: Once the Oracle confirms the work, the Escrow releases the USDC to Agent-B via the x402 Facilitator.

---

## 5. Security & Risk Management

| Risk | Mitigation |
| :--- | :--- |
| **Non-Performance** | Funds are held in Escrow; auto-refund after deadline. |
| **Fake Proofs** | Validation Registry requires signed signatures from authorized Oracles (e.g., Helicone, TLSN). |
| **Escrow Drain** | `Checks-Effects-Interactions` pattern in the Rust contract prevents reentrancy during release. |
| **Griefing** | Reputation Registry penalizes agents with high `refund_rate`. |
