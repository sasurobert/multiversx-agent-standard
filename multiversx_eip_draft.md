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
Manages the "Agent ID". Unlike Ethereum, we utilize **Dynamic SFTs** (Semi-Fungible Tokens) where the `URIs` can be updated, but the `TokenID` remains the identifier.

**Agent ID Logic:**
- **Token**: A specific SFT from the `MX8004-xxxx` collection.
- **Dynamic URI**: Points to the **Agent Registration File (ARF)** (JSON).
- **Attributes**: `[OwnerAddress, ReputationContractAddress, ValidationContractAddress]`

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
- `registerAgent(metadata_uri, endpoints[])`: Mints the Identity SFT.
- `updateAgent(sft_nonce, new_metadata_uri)`: Updates the Dynamic URI.
- `getAgent(sft_nonce)`: Returns full metadata struct.

### 3.2. The Reputation Registry
A contract that aggregates trust signals. It is linked to the Identity Registry.

**Logic:**
- **Feedback**: Users/Contracts sign a "Job Completion" message.
- **Score**: The contract aggregates these signals (potentially using a weighted algorithm based on the rater's own reputation/stake).

**Contract Interface:**
- `submitFeedback(agent_sft_nonce, score, comment_hash, signature)`: Records a review.
- `getReputation(agent_sft_nonce)`: Returns `(total_score, review_count)`.

### 3.3. The Validation Registry
For high-stakes agents. It acts as an "Oracle" for agent integrity.

**Logic:**
- Agents submit "Proofs" (TLSN, zkML, or Optimistic assertions) after a job.
- Validators (or the user) verify these proofs on-chain.

**Contract Interface:**
- `submitProof(job_id, proof_data)`: Agent posts proof.
- `validateProof(job_id, result)`: Validator posts result.
- `isVerified(job_id)`: Returns the verification status.

## 4. Interaction Flow (The "Molt" Pattern)
1.  **Discovery**: User queries Identity Registry for "DeFi" agents.
2.  **Check**: User calls Reputation Registry to filter for `score > 4.5`.
3.  **Engage**: User sends payment to the `session` endpoint defined in the Identity.
4.  **Verify**: (Optional) User checks Validation Registry for the job's proof.

## 5. Rationale
- **Dynamic NFTs**: MultiversX's native mutable attributes allow cheap updates to Agent Metadata without burning/reminting.
- **Decoupled Registries**: Separating Identity, Reputation, and Validation allows for modular upgrades (e.g., swapping the Reputation algorithm without breaking Identity).
