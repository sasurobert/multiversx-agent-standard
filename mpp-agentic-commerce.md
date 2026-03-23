# MPP Integration for Agentic Commerce: Technical Specification

This document outlines the architecture and design specifications for integrating the MultiversX Mobile Payment Protocol (MPP) natively into agentic commerce frameworks, specifically targeting OpenClaw, Moltbot, and the broader Model Context Protocol (MCP) ecosystem.

## 1. Core Philosophy

In an agentic economy, agents must be first-class financial citizens. They need the ability to seamlessly request compensation for their work (acting as servers/receivers) and autonomously pay for resources, data, or services provided by other agents (acting as clients/payers).
MPP serves as the universal language for these transactions, bridging human-to-machine (H2M) and machine-to-machine (M2M) economies.

## 2. Agent Authorization & Spending Mechanisms

Based on risk management and autonomy requirements, agents will utilize a hybrid spending authorization model. Agents will "spend from their own wallet," which can be configured dynamically.

### 2.1 The Agent "Hot Wallet" (Native Wallet)
Every autonomous agent is provisioned with its own native operational wallet (a generated mnemonic/keystore). This is the agent's "hot wallet" used for daily operations and gas fees.

### 2.2 Spending Allowances & Risk Management
To prevent rogue LLM behavior from draining funds, the agent's spending logic intercepts all payment requests and applies policy checks:
*   **Per-Transaction Limit:** Maximum amount allowed for a single 402 challenge (e.g., max 1 USDC).
*   **Daily/Epoch Allowance:** Total budget the agent is allowed to spend autonomously per day.
*   **Approved Tokens:** Whitelist of ESDT tokens the agent is allowed to spend (e.g., USDC, EGLD, WEGLD).
*   **Human-in-the-Loop (Fallback):** If a payment exceeds the autonomous allowance, the agent automatically pivots to a Human-to-Machine flow, generating an MPP URI for the human owner to approve and sign via xPortal.

## 3. Machine-to-Machine (M2M) Flow: Agent as a Client (Payer)

When Agent A (Client) requests a service from Agent B (Server) via an MCP tool call.

### 3.1 Interception Protocol
1.  Agent A invokes a tool on Agent B's MCP server.
2.  Agent B responds with an HTTP `402 Payment Required` (or an MCP protocol equivalent error), including the MPP Challenge in the response payload.
3.  Agent A's networking layer (or Moltbot skill interceptor) catches the 402 error.

### 3.2 Autonomous Settlement
1.  Agent A parses the MPP Challenge using `mppx-multiversx`.
2.  Agent A evaluates the charge against its internal **Spending Allowances**.
3.  If authorized:
    *   Agent A uses its native "Hot Wallet" to sign the transaction specified in the challenge.
    *   The transaction is broadcasted to the MultiversX network (or via the relayed Facilitator).
    *   Agent A obtains the `txHash` or waits for the `SettlementRecord`.
4.  If unauthorized (exceeds limit):
    *   Agent A halts and messages the human owner: *"Tool execution requires 50 USDC, which exceeds my 5 USDC limit. Please authorize via this MPP link: [mpp://...]"*

### 3.3 Retry & Proof of Payment
1.  Agent A repeats the original MCP tool call, this time appending the payment proof.
2.  Proof is attached via MCP metadata, HTTP Headers (`x-mpp-payment: <txHash>`), or as a direct tool argument (`payment_proof: txHash`).

## 4. Machine-to-Machine (M2M) Flow: Agent as a Server (Receiver)

When an Agent (e.g., OpenClaw instance) provides premium data, compute, or actions.

### 4.1 MCP Server Integration (`mcp-mpp-middleware`)
*   **Tool Pricing Metadata:** MCP tool definitions are extended to include pricing.
    ```json
    {
      "name": "generate_deep_research",
      "description": "Generates a 10-page market research report.",
      "x-mpp-price": {
        "amount": "5000000",
        "currency": "USDC-c76f1f"
      }
    }
    ```
*   **Middleware:** An MCP server middleware intercepts incoming requests to priced tools.
*   If no valid payment proof is provided, the middleware generates a native MPP Challenge using the `mppx-multiversx` SDK and throws the 402 error back to the client.

### 4.2 Verification
*   The Agent Server uses the `mpp-facilitator-mvx` (or internal logic) to verify the incoming `txHash` against the issued challenge.
*   Upon successful verification of the blockchain state, the tool execution proceeds and returns the result.

## 5. Human-to-Machine (H2M) Flow Integration

For scenarios where the agent interacts directly with a human user (e.g., via a chat interface like Telegram, Slack, or a web frontend).

1.  **Trigger:** User asks the agent to perform a premium task ("Mint an NFT collection for me").
2.  **Quote:** Agent responds: "I can do that. The cost is 0.5 EGLD for network fees and my service."
3.  **Payment Request:** Agent generates an MPP URL and renders it as a QR code or clickable deep link in the chat interface.
4.  **Listen:** Agent polls the `mpp-facilitator-mvx` webhook/polling endpoint waiting for the `SettlementRecord`.
5.  **Execution:** Once the human signs via xPortal and the transaction settles, the agent automatically resumes the task and replies: "Payment received! Minting your collection now..."

## 6. Implementation Ecosystem Strategy

To make this stand out, we should build these modular components:

1.  **`@multiversx/mcp-payment-middleware`:** A drop-in TypeScript package for MCP server developers to easily wrap their tools with 402 MPP challenge generation and verification.
2.  **Moltbot Payment Skill (`moltbot-mpp-skill`)**: A native skill for Moltbot agents that handles 402 interceptions, allowance checking, and automatic transaction signing.
3.  **OpenClaw Paid Tools Template**: An OpenClaw template showcasing an agent that charges per query or per action using MPP.
