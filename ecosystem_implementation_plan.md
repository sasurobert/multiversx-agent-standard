# Ecosystem Implementation Plan: MultiversX Agent Economy

## 1. Executive Summary
This document outlines the step-by-step roadmap to build a **"Base-Killer" Agent Ecosystem** on MultiversX. The goal is to replicate the success of Moltbook (Base) by providing:
1.  **Shared Infrastructure**: Managed services (MCP, x402, Relayer) that lower the barrier to entry.
2.  **Native OpenClaw Support**: A published "Skill Bundle" that makes any OpenClaw agent compatible with MultiversX instantly.
3.  **MX-8004 Trust Layer**: On-chain verification to prevent the "spam agent" issues seen on other chains.

---

## Phase 1: The Foundation (Shared Infrastructure)
**Goal**: Create the "Serverless" environment for agents.
**Timeline**: Weeks 1-2

### Step 1.1: Deploy Core Services (VPS)
*   **Provision**: Ubuntu VPS (DigitalOcean/AWS), simple Docker setup.
*   **Security**: UFW, Fail2Ban, SSL (Certbot).

### Step 1.2: The Shared MCP Server
*   **Spec**: A Read-Only API endpoint (`https://mcp.molt.bot`) conforming to the Model Context Protocol.
*   **Function**: Proxies requests to MultiversX Observers (Elasticsearch/Gateway).
*   **Key Endpoints**:
    *   `GET /tools/balance?address=...`
    *   `GET /tools/token-price?token=...`
*   **Outcome**: Agents can "read" the chain without running a node.

### Step 1.3: The x402 Facilitator (Payments)
*   **Spec**: An HTTP/Websocket service (`wss://pay.molt.bot`) handling x402 headers.
*   **Function**:
    *   Listens for on-chain transfers/streams to registered agents.
    *   Emits `StreamStarted` events to connected sockets (Agents).
    *   Handles "Settlement" (claiming funds from stream).
*   **Outcome**: Agents receive "Push Notifications" when they get paid.

### Step 1.4: The Molt Relayer
*   **Spec**: A Transaction Broadcaster (`POST https://relay.molt.bot`).
*   **Function**:
    *   Accepts signed transactions from Agents.
    *   Validates signatures against the MX-8004 Registry.
    *   Broadcasts to chain (paying gas).
*   **Outcome**: Agents don't need to manage EGLD for gas initially (Relayer takes a cut or is subsidized).

---

## Phase 2: The Trust Layer (Smart Contracts)
**Goal**: Deploy the MX-8004 Registry Suite.
**Timeline**: Week 3

### Step 2.1: Identity Registry (Rust SC)
*   **Spec**: `MX8004-Identity`.
*   **Logic**:
    *   `register(metadata_uri)`: Mints a **Dynamic SFT/NFT**.
    *   `update(nonce)`: Updates URI.
*   **Attributes**: `[Owner, PublicKey, ReputationSC_Address]`.

### Step 2.2: Reputation Registry (Rust SC)
*   **Spec**: `MX8004-Reputation`.
*   **Logic**:
    *   `submitFeedback(agent_id, score, comment_hash, visual_proof_hash)`.
    *   Calculates "Trust Score" (e.g., weighted by feedback provider's stake).

### Step 2.3: Validation Registry (Rust SC)
*   **Spec**: `MX8004-Validation`.
*   **Logic**:
    *   `submitProof(job_id, proof_data)`.
    *   Stores ZK proofs or TLSN attestation hashes.

---

## Phase 3: The "Brain" (OpenClaw Integration)
**Goal**: Make running a MultiversX agent a "One-Command" experience.
**Timeline**: Week 4

### Step 3.1: The MultiversX Skill Bundle
*   **Repo**: `multiversx-openclaw-skills`.
*   **Contents**:
    *   `SKILL.md`: "Instructions for the AI on how to use MultiversX".
    *   `tools/sign_tx.js`: Uses `mx-sdk-js` to sign payloads.
    *   `tools/query_mcp.js`: Connects to Shared MCP.
    *   `tools/listen_x402.js`: Connects to x402 Websocket.
*   **Action**: Publish to **ClawHub** (`clawhub publish multiversx`).

### Step 3.2: The Starter Kit Repo
*   **Repo**: `moltbot-starter-kit`.
*   **Spec**:
    *   Default `agent.json` (OpenClaw config).
    *   Pre-installed `multiversx-skill`.
    *   `setup.sh`: Generates wallet, registers on MX-8004.
    *   `start.sh`: Launches OpenClaw with MultiversX context.

---

## Phase 4: The Economy (Marketplace & Launch)
**Goal**: connect Humans (Buyers) with Agents (Sellers).
**Timeline**: Week 5+

### Step 4.1: The Marketplace UI (`market.molt.bot`)
*   **Indexer**: Listens for MX-8004 Mint events.
*   **UI**: "App Store" grid of agents.
    *   "Buy Now" button triggers x402 stream via WalletConnect.
*   **Function**: Users browse -> Connect Wallet -> Pay -> Chat opens.

### Step 4.2: Launch Incentives
*   **Bounties**: "$50 in EGLD for first 100 registered agents".
*   **Hackathon**: "Best OpenClaw Skill for MultiversX".

---

## Success Metrics
1.  **Registry**: 100+ Verified Agents on Mainnet.
2.  **Volume**: 10,000+ x402 streaming transactions.
3.  **Adoption**: `multiversx` skill becomes a "Top 10" download on ClawHub.
