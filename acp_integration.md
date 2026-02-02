# Agentic Commerce Protocol (ACP) Integration for MultiversX (Reference Implementation)

## 1. Executive Summary
Unlike x402, the **Agentic Commerce Protocol (ACP)** does not have a central registry of "supported networks". "Official" integration means releasing a **Reference Implementation** or **Adapter** that fully complies with the ACP Spec. By building `mx-acp-adapter` and publishing it, we enable any MultiversX merchant to be ACP-compatible.

## 2. Technical Strategy: The "ACP Adapter"
We will build a standalone open-source repository: `multiversx-acp-adapter`. This serves as the "Official MultiversX Middleware".

### 2.1. Component 1: The Feed Generator (Product Spec)
*   **Goal**: Generate compliant `products.json` from on-chain state.
*   **Implementation**: A TypeScript service using `mx-sdk-js`.
*   **Input**: Smart Contract Address (Marketplace) or Collection ID.
*   **Process**:
    *   Query `GetActiveListings`.
    *   Map `tokenAttributes` -> ACP `description`/`attributes`.
    *   Map `price` -> ACP `price` (handling decimals).
*   **Output**: Hosting a public JSON endpoint.

### 2.2. Component 2: The Checkout Gateway (Checkout Spec)
*   **Goal**: Handle `POST /cart` and `POST /checkout`.
*   **Standard Compliance**:
    *   Must accept ACP standard headers.
    *   Must return ACP standard error codes.
*   **MultiversX Specifics**:
    *   The `/checkout` endpoint must return a **Transaction Payload** wrapped in an ACP "Action".
    *   Since ACP assumes credit cards (Delegated Payment), we will adopt the **"Crypto Payment Extensions"** pattern (returning a payment address and amount, or a WalletConnect URI).

### 2.3. Publishing "Official" Support
Once the adapter is built:
1.  **Docs**: Publish "How to Sell on MultiversX with AI Agents" guide.
2.  **Registry**: Submit our adapter to any community lists (e.g., `awesome-agentic-commerce` if it exists).
3.  **OpenAI**: Configuring a GPT to use this API is the final validation.

## 3. Implementation Plan (The Code)
We will create a monorepo `multiversx-acp-kit` containing:
1.  `packages/spec`: TypeScript types for ACP on MultiversX.
2.  `packages/server`: A Node.js (NestJS) server that wraps any interaction into ACP endpoints.
    *   `POST /v1/checkout` -> Returns `tx_data` for the agent to sign.

## 4. Verification
*   **Validation**: Use the [Agentic Commerce Validator](https://agenticcommerce.dev) (if available) or manual curl tests against the spec.
*   **Live Test**: Create a Custom GPT that points to our deployed Adapter URL and ask it to "Buy the NFT".
