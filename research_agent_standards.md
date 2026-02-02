# Research: Agent Standards & Architectures

## Executive Summary
To build a scalable "Agent Economy" on MultiversX, we need three distinct layers of standardization:
1.  **Social/Trust Layer**: Who is this agent? (ERC-8004 -> **MX-8004**)
2.  **Economic Layer**: How does it get paid? (**x402** / UCP)
3.  **Functional Layer**: How does it think/act? (**MCP**)

## 1. The Trust Standard: Adapting ERC-8004 for MultiversX
ERC-8004 ("Smart Layer") allows agents to discover and trust each other. We can port this to MultiversX as **MX-8004** using native features.

### A. Identity Registry (The "Agent Card")
- **Standard**: Agents must hold a specific **SFT (Semi-Fungible Token)**.
- **Metadata**: The SFT contains the `AgentCard` JSON URI (name, description, capabilities, endpoints).
- **Validation**: Only agents holding this SFT are "Verified."
- **MultiversX Advantage**: SFTs are native and cheap to mint/transfer.

### B. Reputation Registry
- **Mechanism**: A smart contract that tracks "Success Rate."
- **Data**: Other agents or users sign messages attesting to completion of jobs.
- **Storage**: Aggregated scores on-chain; detailed logs off-chain (IPFS/Arweave), linked via contract.

### C. Validation Registry (The "Credit Score")
- **Mechanism**: Hooks for 3rd party validators (e.g., the Verifiable Inference Oracle) to post "Proofs of Integrity."

## 2. The Functional Standard: Model Context Protocol (MCP)
**MCP** is the "USB-C for AI." It standardizes how an agent connects to tools (databases, APIs, wallets).

- **Role in MultiversX**:
    - Create a **"MultiversX MCP Server"**: A standard Reference Implementation that allows *any* MCP-compliant LLM (Claude, etc.) to:
        1.  Read Wallet Balances.
        2.  Sign Transactions (via secure enclave/custodian).
        3.  Query Smart Contract State.
    - **Standardization**: Define a `multiversx-mcp` specification so all agents use the same tool definitions (`get_account`, `send_tx`, `verify_agent_identity`).

## 3. The Economic Standard: x402 + UCP
- **x402 (Streaming)**: Standardize the "Pay-per-Token" stream. Agents typically require a deposit or an active stream to begin inference.
- **UCP (Unified Commerce Protocol)**: Defines the invoice and order lifecycle.

## The "MultiversX Agent Standard" (MAS) Proposal
An agent is compliant if:
1.  **Identity**: Holds an MX-8004 SFT.
2.  **Interface**: Exposes an MCP-compatible endpoint (or acts as an MCP client).
3.  **Payment**: Acceptance of x402 streams for services.
4.  **Verification**: (Optional) Provides TLSN proofs for high-value outputs.
