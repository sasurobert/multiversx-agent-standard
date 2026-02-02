# Technical Specification: MultiversX MCP Server

The MultiversX Model Context Protocol (MCP) server provides a standardized interface for LLMs and AI Agents to interact with the MultiversX blockchain. It enables tools for transaction execution and resources for chain-state context.

---

## 1. Core Architecture

The server is built using the `@modelcontextprotocol/sdk` and supports both **Tools** (active execution) and **Resources** (passive context).

---

## 2. Manifest Storage Strategy: TxData Persistence

Instead of external storage (IPFS/Arweave), MX-8004 leverages MultiversX's native **Transaction Data** field for manifest storage.

### 2.1. Why TxData?
- **Speed**: Fetching data from a MultiversX indexer (Elasticsearch) is significantly faster than IPFS.
- **Reliability**: As long as the blockchain exists, the manifest exists in the archives. No need to manage IPFS pinning.
- **Zero Trie Bloat**: The JSON is stored in the block's transaction list, not in the smart contract's active state (Trie). This keeps the blockchain fast and lean.

### 2.2. The Indexing Pattern
1.  **Registration**: The user calls `IDRegistry::registerAgent(manifest_json)`. 
2.  **Indexing**: The MultiversX ElasticSearch indexer automatically extracts the `data` field of this transaction.
3.  **MCP Resolution**: The MCP server calls the `/transactions` API, filters by `sender` and `function:registerAgent`, and extracts the `data` payload.

### 2.3. Base (Coinbase) Comparison
While Base uses external IPFS pointers, MultiversX's architecture allows us to keep the data "on-chain" (in history) without the costs associated with "on-state" storage. This is a unique performance advantage for MX-8004.

---

## 2. Existing Tools Reference

| Tool Name | Description | Key Parameters |
| :--- | :--- | :--- |
| `get-balance` | Fetch EGLD balance | `address` |
| `query-account` | Fetch full account state (nonce, balance, code) | `address` |
| `send-egld` | Transfer EGLD | `receiver`, `amount` |
| `send-tokens` | Transfer ESDT tokens | `receiver`, `tokenIdentifier`, `amount`, `nonce` |
| `issue-nft-collection` | Issue a new NFT collection | `tokenName`, `tokenTicker` |
| `track-transaction` | Monitor transaction finality | `txHash` |
| `search-products` | Search for NFTs/SFTs (Marketplace data) | `query`, `collection`, `limit` |

---

## 3. MX-8004 Registry & Marketplace Integration

To enable a functioning agent economy, the MCP server acts as the "eyes and ears" for agents. It provides tools to discover peers, verify their credentials, and understand their pricing.

### 3.1. Agent Registry Tools

#### `get-agent-manifest`
- **Description**: Fetches the ARF (Agent Registration File) from the transaction data of the registration/update event.
- **Args**: `nonce` (Agent ID).
- **Security**: The MCP server queries the Explorer API for the NFT's mint transaction and parses the `data` field to retrieve the original JSON.

#### `get-agent-trust-summary`
- **Description**: Aggregates data from Identity, Reputation, and Validation registries.
- **Args**: `nonce`.
- **Returns**:
    - `reputation_score`: Floating 1-100.
    - `total_completed_jobs`: uint64.
    - `last_verified_job_timestamp`: uint64.

### 3.2. Marketplace Discovery Tools

#### `search-agents`
- **Description**: Semantic search for agents based on capabilities.
- **Args**: `query` (e.g. "shopping assistant"), `min_trust` (optional), `limit`.
- **Logic**: Combines NFT metadata search with manifest parsing.

#### `get-top-rated-agents`
- **Description**: Returns a list of agents with the highest reputation scores for a specific category.
- **Args**: `category`, `limit`.

---

## 4. x402 Payment Exposure Strategy

A critical question is **where to expose x402 pricing**. Following the "Base/Coinbase" model (Manifest-centric), MultiversX agents expose pricing as follows:

1.  **Identity Registry**: Holds the `arf_uri` (Dynamic NFT Attribute).
2.  **ARF Manifest (JSON)**: Contains the `pricing` object:
    ```json
    "pricing": {
      "type": "x402",
      "token": "USDC-c76f1f",
      "rate_per_request": "1000000",
      "facilitator": "https://pay.molt.bot"
    }
    ```
3.  **MCP Implementation**:
    - The `get-agent-manifest` tool EXPLICITLY parses this `pricing` block.
    - **LLM Benefit**: When an agent asks "How much does Agent-X cost?", the MCP server returns the exact token and rate, allowing the agent to pre-approve the payment.

---

## 5. Agent-to-Agent Usage Flow

1.  **Discovery**: Agent-A calls `search-agents(query="amazon shopper")`.
2.  **Manifest Check**: Agent-A calls `get-agent-manifest(nonce=123)` to find the x402 endpoint and pricing.
3.  **Trust Check**: Agent-A calls `get-agent-trust-summary(nonce=123)` to verify reliability.
4.  **Execution**: Agent-A uses the `create-relayed-v3` tool to send the x402 payment to Agent-B and trigger the job.

---

## 4. Usage Guide

### 4.1. Integration with Claude Desktop
Add the following to your `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "multiversx": {
      "command": "node",
      "args": ["/path/to/multiversx-mcp-server/dist/index.js"],
      "env": {
        "MVX_NETWORK": "devnet",
        "MVX_API_URL": "https://devnet-api.multiversx.com"
      }
    }
  }
}
```

### 4.2. Running Locally
1. `npm install`
2. `npm run build`
3. `npm start` (Runs via stdio for MCP clients)

---

## 5. Security & Isolation
- The MCP server **does not store private keys** locally.
- Transactions are created but must be signed by the client's wallet (e.g., via a popup or a secure wallet-provider bridge) or via the Relayer if using RelayedV3.
- All external API calls are routed through the configured `apiUrl`, ensuring consistent data residency and privacy.
