# Technical Specification: MultiversX UCP Integration (MCP Server)

## 1. Executive Summary
This specification details the architecture for the **MultiversX Model Context Protocol (MCP) Server**. This server acts as the bridge between UCP-compliant AI Agents (Google Gemini, Claude, etc.) and the MultiversX Blockchain.

**Business Goal**: Achieve "Official Support" status in the UCP/MCP registries and enable native agentic commerce on MultiversX.
**Success Metric (6 Months)**: Successful listing in the MCP Server Registry and verified end-to-end "Buy/Track" flows for top MultiversX marketplaces.

## 2. Problem Statement
AI agents currently lack a standardized way to discover and purchase assets on the MultiversX blockchain. fragmentation between different marketplace smart contracts prevents agents from providing a unified "shopping" experience.

## 3. Architecture & Functional Vision
The solution is a standalone **TypeScript MCP Server**.

### 3.1. Passive Indexer Strategy (Whitelisted)
The server will automatically index listings but use a **Registry of Trusted Marketplace Contracts** to ensure authenticity.
*   **Whitelisting**: Only contracts successfully reviewed via PR or known by the core team are indexed.
*   **Reputation**: Metadata for each tool call result will include the "Trust Level" of the source contract.

### 3.2. Agentic Workflow (The "Happy Path")
1.  **Search**: User asks agent to find an item. Agent calls `search_products`.
2.  **Product Selection**: User selects an item.
3.  **Buy Initiation**: Agent calls `create_purchase_transaction`.
4.  **Interactive Signing**: The tool returns a standardized **JSON Transaction Object**. The agent hands this to an "Interactive Wallet Tool".
5.  **Tracking**: After broadcast, the agent uses `track_order` to monitor status.

## 4. Requirement & Tool Schemas

### 4.1. Server Architecture: Hybrid (MCP + HTTP)
To serve both **AI Agents** and **Google Merchant Center**, the server will run as a Dual-Mode service:
-   **MCP Interface** (Stdio/SSE): Exposes Tools/Resources to Claude/Gemini.
-   **HTTP Interface** (Fastify/Express): Exposes `GET /feed.json` for Google Merchant Center ingestion.

### 4.2. Tool: `search_products`
**Input**: Keywords, Collection, Price Range.
**Logic**:
-   **Passive Index**: Listens for `UCPListing` events.
-   **Metadata Enrichment**: Performs an async chain query (VmQuery or API) to fetch `URIs`, `Names`, and `Attributes` using the `TokenIdentifier` + `Nonce` from the log. It does **not** expect full metadata in the log event.

### 4.3. Tool: `create_purchase_transaction`
**Input**: `token_identifier`, `nonce`, `quantity`.
**Output**: Standard MultiversX Transaction JSON.

### 4.4. Tool: `track_order`
**Input**: `transaction_hash`.
**Logic**: 
- **Stateless Operation**: Queries MultiversX API.
- **Verification**: Checks `LogEvent.address` against the Whitelist.

## 5. System Architecture & Data Model
- **Product ID**: Represented as `TokenIdentifier-Nonce` for simplicity and ecosystem alignment.
- **Verification Layer**: The MCP server maintains a `whitelists.json` (extendable via PR). During log parsing, it cross-references the `LogEvent.address` with this whitelist.
- **Extensibility**: Merchants can submit PRs to define custom log-to-product mapping logic if they deviate from the "Standard Event" format.

## 6. Resilience & Scalability Strategy

### 6.1. API Gateway vs Custom Node
To prevent rate-limiting and ensure reliability:
- **Default Mode (Global Gateway)**: The MCP server defaults to a dedicated "MCP Gateway" (a load-balanced cluster of Observers) optimized for these query patterns.
- **Custom Mode**: Advanced users can configure `MVX_API_URL` to point to their own Observer Node or commercial API provider (e.g., Blast, Tatum) to avoid shared limits.

### 6.2. Concurrency & Slippage
- **Blockchain Native Safety**: We rely on the atomic nature of the blockchain. If an item is sold before the user signs:
    - The Smart Contract `buy` function will fail (revert).
    - The funds are never transferred (or instantly returned).
- **Metadata**: The `search_products` tool returns a `last_updated` timestamp. Agents should warn users if a listing is stale (>1 minute old).

### 6.3. Hallucination Prevention
- **Pending States**: If `track_order` is called immediately after signing, the API might return 404. The Tool must catch this and return `status: pending, retry_after: 5s` instead of "Failed".

## 7. Implementation Plan
- **Milestone 1**: Project scaffold and `search_products` for whitelisted contracts.
- **Milestone 2**: Transaction generation for standard `buy` calls.
- **Milestone 3**: Order tracking logic parsing on-chain events.
- **Milestone 4**: Submission to the Model Context Protocol global registry.
