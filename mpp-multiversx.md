# MPP MultiversX Integration Specification (MX-8004)

This specification defines the Machine Payments Protocol (MPP) implementation for the MultiversX ecosystem, extending the fundamental IETF draft standards for autonomous AI agent commercialization.

## 1. Executive Summary

This architecture establishes a hyper-efficient, 100% on-chain solution for handling MPP `charge` and `session` intents. Aligning with the native capabilities of the MultiversX blockchain, we implement a **Data Payload Tagging Strategy**.

Unlike overcomplicated Treasury Smart Contracts (which waste storage gas) or Derived HD Wallet Addresses (which require expensive gas sweeps), Data Payload Tagging requires the AI Agent to simply append the `challenge.id` into the `data` field of a standard EGLD/ESDT transfer sent to the Facilitator's primary wallet. This inherently solves idempotency and tracking while concentrating all liquidity immediately into a single, fully-verifiable wallet.

## 2. Component Architecture

To support a fully-featured, production-ready MPP integration on MultiversX, two primary components must be developed:

### 2.1. `mpp-facilitator-mvx` (NestJS Gateway/Proxy)
The API gateway that AI agents interact with directly to consume services.
- **Role:** Intercepts unauthenticated HTTP requests, provides its single deposit receiver address, issues `402 Payment Required` challenges using the `multiversx` method, and verifies incoming MPP credentials against the MultiversX blockchain.
- **Responsibilities:**
  - Generate `WWW-Authenticate: mpp` challenge parameters.
  - Expose `/.well-known/agent-card.json` and `/openapi.json` for AI agent discovery.
  - Validate `txHash` submitted by agents by querying the MultiversX node/API to confirm receipt, amount, and exact data payload matching the challenge ID.
  - Manage idempotency by ensuring a validated `txHash` is only consumed once per challenge string.
  - Broadcast Relayed V3 transactions on behalf of agents lacking native gas.

### 2.2. `mppx-multiversx` (TypeScript SDK)
The exact implementation of the `mppx` interface for MultiversX agents.
- **Role:** Empowers AI agents to seamlessly handle MPP challenges natively.
- **Responsibilities:**
  - Implements the `multiversx` payment method logic.
  - Parses MPP headers.
  - Uses `@multiversx/sdk-js` to correctly format transactions, critically injecting the `mpp:<challenge_id>` string into the `data` payload of the transfer.
  - Handles NativeAuth signature processing for `Authorization: Payment`.

---

## 3. MPP Protocol Conformance

This section defines how MultiversX implements every core requirement of the MPP specification natively.

### 3.1. Core HTTP Semantics
Any request lacking valid payment credentials will be met with a strict `402 Payment Required` HTTP response. The `address` is the Facilitator's primary deposit wallet.
```http
HTTP/1.1 402 Payment Required
WWW-Authenticate: mpp challenge="ch_1234abc", method="multiversx", intent="charge", amounts="1000000000000000000", tokens="EGLD", address="erd1fac..."
```

### 3.2. MPP Intents
The protocol abstracts payment patterns. MultiversX supports all proposed intents natively via Data Tags.

#### 3.2.1. The `charge` Intent (One-time payments)
Used for single API calls or atomic resource consumption.
1. **Flow:**
   - Agent receives a `-charge-` challenge containing the single Facilitator `address` and a `challenge.id` (e.g., `ch_1234abc`).
   - Agent sends a standard EGLD/ESDT transfer to the `address` with `Data: mpp:ch_1234abc`.
   - Agent submits the resulting `txHash` in the `Authorization: mpp` header.
2. **Facilitator Validation:** The `mpp-facilitator-mvx` fetches the `txHash` from the MultiversX API, confirming the transaction was successful, targets the Faciliator `address`, transfers the required amount, and specifically holds the `mpp:ch_1234abc` data payload.
3. **Idempotency:** The Facilitator guarantees that the combination of `txHash` and `challenge.id` is only marked as paid exactly once in its database. Because the chain is immutable, the user can instantly prove they paid exactly for that ID.

#### 3.2.2. The `session` Intent (Prepaid/Metered access)
Used for continuous access without a per-request transaction delay.
1. **Flow:**
   - Agent receives a `-session-` challenge with a `challenge.id` (e.g., `sess_789xyz`).
   - Agent pre-funds the Facilitator `address` with a bulk deposit transaction using `Data: mpp_sess:sess_789xyz`.
   - For subsequent API calls, the Agent provides the `Authorization` tying them to the session.
2. **Facilitator Validation:** The Proxy validates the initial funding `txHash` and tracks the remaining decrementing balance of that session locally.

#### 3.2.3. The `subscription` Intent (Recurring payments)
Used for distinct time-bound or recurring thresholds, functioning identically to `session` but validated against temporal epochs rather than byte throughput (e.g., `Data: mpp_sub:monthly_tier`).

### 3.3. MPP Methods
This specification strictly defines the `multiversx` payment method.

- **Method Identifier:** `multiversx`
- **Challenge Parameters Required:**
  - `address`: The Facilitator's receiver wallet.
  - `tokens`: Token identifier (e.g., `WEGLD-123456`, `EGLD`).
  - `amounts`: Value represented in the token's lowest denomination.
  - `chainId`: Network identifier (`1` for Mainnet, `D` for Devnet, `T` for Testnet).

### 3.4. MPP Extensions

#### 3.4.1. Discovery (Agent Cards & OpenAPI)
The Facilitator MUST host a complete API blueprint. AI agents fetch `/.well-known/agent-card.json` to understand the Facilitator's capabilities.
```json
// /.well-known/agent-card.json
{
  "name": "Tempo MultiversX Analysis Agent",
  "description": "Provides data points using MPP over MultiversX",
  "openapi": "https://api.example.com/openapi.json",
  "payment": {
    "protocol": "mpp",
    "methods": ["multiversx"],
    "intents": ["charge", "session"]
  }
}
```

#### 3.4.2. Identity & Authentication (NativeAuth)
To bind the payment to the exact requesting agent and prevent front-running, the MultiversX integration utilizes **NativeAuth**.
- Agents sign a NativeAuth token that includes the HTTP method, route, and `challenge.id`.
- The `Authorization` header combines the MPP payload and NativeAuth token.

#### 3.4.3 Relayed Transactions
For scenarios where an AI agent only holds ESDT and no EGLD (or doesn't want to manage gas), the `mppx-multiversx` SDK supports **Relayed V3 Transactions**.
- The Agent constructs the standard transfer transaction to the Facilitator `address` (including the data tag), signs it, and passes it to the `mpp-facilitator-mvx` inside a proprietary header (`x-mpp-relayed-tx`).
- The Facilitator wraps it in a Relayed V3 transaction, paying the EGLD gas fee roughly recovering it from the ESDT collected, and broadcasts it to the network.
