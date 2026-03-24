# Machine Payments Protocol (MPP) on MultiversX: Reasoning & Architecture

This document outlines the conceptual architecture and core philosophy of the Machine Payments Protocol (MPP) integration for the MultiversX ecosystem, specifically focusing on agentic commerce and M2M (Machine-to-Machine) value exchange.

## 1. Core Philosophy: Agents as Financial Citizens

In a sovereign agentic economy, AI agents must transcend being mere "automated shoppers" and become **first-class financial citizens**. This requires:
- **Autonomy:** The ability to discover, negotiate, and execute payments without constant human intervention.
- **Identity:** A verifiable on-chain footprint via MIP-8004 (Identity Registry).
- **Reputation:** A history of validated job completions and feedback (Reputation Registry).
- **Security:** Programmable constraints (Allowances/Hot Wallets) to mitigate risk.

MPP serves as the universal language for these interactions, bridging Human-to-Machine (H2M) and Machine-to-Machine (M2M) economies.

## 2. Agentic Commerce Strategy

The MultiversX implementation of MPP prioritizes **Gasless Operations** and **Non-Interactive Flows**.

### 2.1 The "Pay-as-you-go" Vision
Agents operating in an MCP (Model Context Protocol) environment frequently interact with "Priced Tools." Instead of monolithic subscriptions, MPP enables a granular, utility-based model where:
1. **Client Agent** requests a specialized action (e.g., "Deep Research").
2. **Server Agent** returns a `402 Payment Required` with an MPP Challenge.
3. **Client Agent** settles the challenge autonomously using its allowance.
4. **Tool Execution** proceeds upon on-chain (or facilitator-relayed) settlement.

### 2.2 Streaming & State Channels (MPP Sessions)
For high-frequency or long-running tasks (e.g., continuous data streams or recurring agent-to-agent sub-tasks), MPP utilizes **Sessions**:
- **Escrow-first**: Funds are locked in a smart contract.
- **Voucher-based**: Micro-payments are authorized off-chain via signed "Vouchers."
- **Efficiency**: Only the final settlement or periodic checkpoints occur on-chain, minimizing gas and latency.
- **Non-Custodial**: Neither the facilitator nor the payee can claim more than the cryptographically authorized total.

### 2.2 Spending Authorization Models
Agents operate under a hybrid security model:
- **Agent "Hot Wallet":** A dedicated operational wallet for autonomous micro-payments.
- **Spending Policies:** Hard limits on per-transaction and daily budgets.
- **H2M Fallback:** If a payment exceeds the agent's autonomous limit, it pivots to an H2M flow, generating a deep-link for the human owner to sign via xPortal.

## 3. Protocol Intents & Mapping

MPP on MultiversX maps to the following standard intents defined in the broader protocol:

| Intent | MultiversX Implementation |
| :--- | :--- |
| **Discovery** | OpenAPI extensions for priced tools and capability discovery. |
| **Authorization** | NativeAuth request binding and secure challenge issuance. |
| **Settlement** | Native ESDT transfers via Data Payload Tagging (`mpp:<challenge_id>`). |
| **Verification** | Validation Registry (MIP-8004) tracking verifiable job completions. |
| **Feedback** | Reputation Registry tracking successful/failed M2M interactions. |

## 4. Interaction Flows

### 4.1 Machine-to-Machine (M2M)
When a Moltbot or OpenClaw agent invokes a tool on an MCP server:
1. **Interception:** The client-side skill intercepts the 402 error.
2. **Evaluation:** The agent checks if the cost (e.g., in USDC) is within its daily allowance.
3. **Settlement:** The agent signs and broadcasts the transaction directly.
4. **Resumption:** The agent retries the tool call with the `txHash` as evidence.

### 4.2 Human-to-Machine (H2M)
When an agent requires payment from a human user:
1. **Request:** The agent generates an MPP URI (e.g., `mpp://...`).
2. **Rendering:** The URI is displayed as a QR code or dApp link in the UI.
3. **Approval:** The user signs via xPortal.
4. **Acknowledge:** The agent polls the facilitator for the `SettlementRecord` and proceeds.

## 5. Security & Trust Architecture

The protocol relies on the **MX8004 Registry System**, which provides a decentralized "Trust Anchor":
- **Identity Registry:** Links agent public keys to metadata and owners.
- **Validation Registry:** Records hashes of verifiable work results.
- **Reputation Registry:** Aggregates feedback into queryable trust scores.

By combining these, we enable a robust, permissionless marketplace where agents can reliably interact with unknown counterparts with cryptographically verifiable trust.
