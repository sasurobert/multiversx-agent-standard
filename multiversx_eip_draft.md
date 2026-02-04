# MIP-8004: MultiversX Agent Standard (Trustless Agents)

## 1. Abstract
MX-8004 is a standard for **Autonomous Agent Identity, Discovery, and Interoperability** on MultiversX. It mirrors the ERC-8004 architecture, defining three core pillars: **Identity** (Dynamic NFT), **Reputation** (Feedback Registry), and **Validation** (Task Verification). This standard enables a trustless "Agent Economy" where agents can be discovered, hired, and verified on-chain.

## 2. Motivation
To enable a scalable ecosystem of AI Agents, we need a standardized way to:
1.  **Identify** agents uniquely across the network.
2.  **Discover** their capabilities (endpoints, pricing, models).
3.  **Verify** their performance and integrity.
MIP-8004 achieves this by porting the proven ERC-8004 logic to MultiversX's sharded architecture, leveraging native ESDT/NFT features for efficiency.

## 3. Specification

The standard consists of three coupled Smart Contracts/Modules.

### 3.1. The Identity Registry (Core)
Manages the "Agent ID". We utilize **Soulbound NFTs** (Non-Fungible Tokens) where the attributes can be updated, but the `TokenID` and `Nonce` remain the permanent identifier.

**Agent ID Logic:**
- **Token**: A specific NFT from the `MX8004-xxxx` collection.
- **Dynamic Attributes**: The NFT stores the `AgentDetails` struct in its attributes.
- **Soulbound**: The contract retains the `transfer` role, ensuring the identity cannot be sold or moved from the owner's address.

**JSON Schema (ARF):**
Stored off-chain (IPFS/Arweave) for gas efficiency.
```json
{
  "name": "Moltbot-01",
  "description": "DeFi Arbitrage Agent",
  "endpoints": {
    "mcp": "https://agent.api/mcp",   // Model Context Protocol
    "session": "https://agent.api/pay", // x402 Payment Stream
    "a2a": "https://agent.api/negotiate" // Agent-to-Agent
  },
  "capabilities": ["defi-swap", "token-analysis"],
  "pricing": { ... },
  "version": "1.0.0"
}
```

**Contract Interface:**
- `register_agent(name, uri, public_key)`: Mints the Identity SFT with all required metadata.
- `update_agent(nonce, new_uri, new_public_key)`: Updates the agent's URI and public key.
- `get_agent(nonce)`: Returns full `AgentDetails` struct.
- `get_agent_id(address)`: Returns the nonce for a registered agent by owner address.

### 3.2. The Reputation Registry
A contract that aggregates trust signals. It is linked to the Identity Registry.

**Logic:**
- **Feedback**: Users/Contracts sign a "Job Completion" message.
- **Score**: The contract aggregates these signals (potentially using a weighted algorithm based on the rater's own reputation/stake).

**Contract Interface:**
- `submit_feedback(job_id, agent_nonce, rating)`: Records a review for a verified job.
- `authorize_feedback(job_id, client)`: Agent allows a specific employer to provide feedback.
- `reputation_score(agent_nonce)`: Returns the current weighted average.
- `total_jobs(agent_nonce)`: Returns the count of reviewed jobs.

### 3.3. The Validation Registry
For high-stakes agents. It acts as an "Oracle" for agent integrity.

**Logic:**
- Agents submit "Proofs" (TLSN, zkML, or Optimistic assertions) after a job.
- Validators (or the user) verify these proofs on-chain.

**Contract Interface:**
- `init_job(job_id, agent_nonce)`: Employer creates a job entry.
- `submit_proof(job_id, proof_data)`: Agent posts proof hash/URI.
- `verify_job(job_id, status)`: Oracle/Owner finalizes the job state.
- `is_job_verified(job_id)`: Returns the verification status.

## 4. Interaction Flow (The "Molt" Pattern)
1.  **Discovery**: User queries Identity Registry for "DeFi" agents.
2.  **Check**: User calls Reputation Registry to filter for `score > 4.5`.
3.  **Engage**: User sends payment to the `session` endpoint defined in the Identity.
4.  **Verify**: (Optional) User checks Validation Registry for the job's proof.

## 5. Rationale
- **Dynamic NFTs**: MultiversX's native mutable attributes allow cheap updates to Agent Metadata without burning/reminting.
- **Decoupled Registries**: Separating Identity, Reputation, and Validation allows for modular upgrades (e.g., swapping the Reputation algorithm without breaking Identity).
