# Technical Specification: MX-8004 (MultiversX Agent Standard)

This document provides a deep-dive specification for the **MX-8004** standard on MultiversX. It detail every endpoint, storage mapper, and event across the three core registries, with explicit focus on **Access Control**, **Security**, and **Authenticity**.

---

## 1. Identity Registry Contract
**Role**: The primary directory for AI Agents. Handles the lifecycle of Agent Identities via Dynamic NFTs.

### 1.1. Storage Mappers
```rust
/// The TokenID of the Agent NFT collection (e.g., AGENT-123456)
#[storage_mapper("agentNftTokenId")]
fn agent_nft_token_id(&self) -> SingleValueMapper<TokenIdentifier>;

/// The URI of the Agent Registration File (ARF)
#[storage_mapper("agentUri")]
fn agent_uri(&self, nonce: u64) -> SingleValueMapper<ManagedBuffer>;

/// ECIES Public Key for encrypted messaging
#[storage_mapper("agentPublicKey")]
fn agent_public_key(&self, nonce: u64) -> SingleValueMapper<ManagedBuffer>;
```

### 1.2. Endpoints & Logic

#### `registerAgent`
- **Logic**: Mints a new NFT. The full **Manifest JSON** is passed in the transaction `data` field. The contract stores the agent's name and public key in its local state for fast resolution.
- **Access Control**: **Public**. Anyone can register.
- **Authenticity**: The manifest is permanently linked to the registration transaction in the blockchain history.

#### `updateAgent`
- **Logic**: Updates the `public_key` in state. The updated **Manifest JSON** is passed in the transaction `data` field.
- **Access Control**: **NFT Owner Only**.
- **Security**: The most recent `updateAgent` transaction for a specific `AgentID` is considered the active manifest by the MCP server.

### 1.3. Events
- `agentRegistered(owner, nonce, Name, ARF_URI)`: Emitted upon successful minting.
- `agentUpdated(nonce, New_URI)`: Emitted when profile is modified.

---

## 2. Reputation Registry Contract
**Role**: A trust aggregator that prevents "Sybils" from faking intelligence or uptime.

### 2.1. Logic & Security
Authenticity is solved via **Cross-Contract Verification**. The Reputation Registry only accepts feedback for jobs that have been cryptographically verified by the `Validation Registry`.

### 2.2. Endpoints

#### `submitFeedback`
- **Logic**: Increments the `totalJobs` and updates the floating-point `reputationScore` for the agent.
- **Access Control**: **User (Job Employer)**.
- **Security Check**:
    1.  The contract calls `ValidationRegistry::is_job_verified(job_id)`.
    2.  If `false`, the transaction panics.
    3.  This ensures that ratings can ONLY be given for completed, verified tasks.

### 2.3. Events
- `reputationUpdated(agent_nonce, new_score)`: Emitted whenever a rating is processed.

---

## 3. Validation Registry Contract
**Role**: The "Judge" that verifies if an AI agent actually performed the task it was paid for.

### 3.1. Logic & Security
This is the most critical security component. Authenticity is achieved via **Proofs of Execution** (e.g., TLSN, ZK, or Trusted Oracle signatures).

### 3.2. Endpoints

#### `submitProof`
- **Logic**: Stores a hash of the result data.
- **Access Control**: **Agent (NFT Owner)**. Only the agent assigned to the job can submit proof.
- **Authenticity**: Verified by checking that the caller owns the `AgentID` associated with the `job_id`.

#### `verifyJob`
- **Logic**: Finalizes the job state to `Verified`.
- **Access Control**: **Oracle/Validator Set Only**.
- **Security**: Only authorized Oracles (e.g., a Helicone Gateway or a ZK-Verifier contract) can move a job from `Pending` to `Verified`.

### 3.3. Events
- `jobVerified(job_id, agent_nonce, status)`: Emitted once the Oracle confirms the proof.

---

## 4. ACP Integration: The Commerce Engine
ACP extends MX-8004 with **On-Chain Escrow** and **Buyer Mode** functionality. For high-value tasks, the Reputation and Validation registries are linked to the **ACP Escrow Contract** to ensure payment is only released upon verified job completion.

---

## 5. Security Summary Table

| Interaction | Access Control | Security Mechanism |
| :--- | :--- | :--- |
| **Registration** | Public | Native NFT ownership (Nonce-based) |
| **Profile Update** | NFT Owner | E6 Signature + Ownership check |
| **Proof Submission** | NFT Owner | Job-to-Agent mapping verification |
| **Job Verification** | Authorized Oracle | Multi-sig or ZK-Proof verification |
| **Reputation** | Task Employer | Requires Job ID verification flag |

---

## 5. Interaction Flow (Authenticity Loop)
1. **User** sends USDC -> **Agent**.
2. **Agent** performs work -> **Validation SC** (Submit Proof).
3. **Oracle** verifies work -> **Validation SC** (Mark Verified).
4. **User** sees Verified -> **Reputation SC** (Submit Rating).
5. **Reputation SC** checks **Validation SC** -> Updates Score.

This loop ensures that **Authenticity** is mathematically and cryptographically linked across all three registries.
