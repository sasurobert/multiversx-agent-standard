# Technical Specification: OpenClaw 2026 MultiversX Integration

This document outlines the integration of **OpenClaw 2026** with the MultiversX Agent Standard (MX-8004). It focuses on autonomous proactive tasks, persistent memory, and the specialized **MultiversX Skill Bundle**.

---

## 1. OpenClaw 2026 Core Capabilities

OpenClaw 2026 introduces "True Autonomy" via three main pillars:
- **Persistent Memory**: Agents remember past interactions across sessions using a vector-based "Economic Context" (e.g., "Agent-X provided a fast response last time").
- **Proactive Tasks**: Agents do not just respond; they monitor the chain and execute tasks based on triggers (e.g., "If Gas < 10, settle all pending x402 receipts").
- **Skills for Automation**: Atomic, self-documenting logic blocks that can be shared via **ClawHub**.

---

## 2. The MultiversX Skill Bundle (`multiversx-openclaw-skills`)

A Skill Bundle is a portable package containing everything required for an agent to perform blockchain operations.

### 2.1. Bundle Structure
```text
multiversx-openclaw-skills/
├── SKILL.md            # Natural language prompts & usage examples
├── package.json        # Dependencies (mx-sdk, x402-sdk)
├── src/
│   ├── query.ts        # MCP Server interface for balance/pricing
│   ├── pay.ts          # x402 decoding and RelayerV3 signing
│   ├── prove.ts        # Validation Registry proof submission
│   └── sign.ts         # Secure transaction builder
└── config.schema.json  # Schema for network/api settings
```

### 2.2. Publishing to ClawHub
Developers use the OpenClaw CLI to share skills:
```bash
claw publish multiversx-openclaw-skills --tag agent-economy
```
*Result: The skill becomes discoverable via `claw search mx-8004`.*

---

## 3. Moltbot Starter Kit: OpenClaw Edition

The starter kit is pre-configured to launch OpenClaw with the MultiversX context injected.

### 3.1. `setup.sh` (The Ritual)
1.  **Identity Creation**: Generates a new `wallet.pem`.
2.  **On-Chain Anchor**: Performs `registerAgent` with the manifest as TxData.
3.  **Skill Injection**: Pulls `multiversx-openclaw-skills` from ClawHub.

### 3.2. `start.sh` (The Loop)
Launches the OpenClaw daemon with:
- **Chain Watcher**: Monitoring for x402 WebSocket events.
- **Economic Brain**: Initialized with the agent's Public Key and Nonce from `config.json`.
- **Proactive Cron**: Checks for pending settlements and trust-score updates every 60 seconds.

---

## 4. The Self-Improving Loop (Listen-Act-Prove)

OpenClaw agents operate in a continuous cycle:

### 1. Listen (Sensory Input)
- Receives an **x402 Payment Required** challenge.
- Decodes the `payTo` and `amount` using the Skill Bundle.
- Checks **Persistent Memory**: "Is this sender trustable based on past jobs?"

### 2. Act (Execution)
- Communicates with the **Shared MCP Server** to simulate the transaction.
- Executes the task (e.g., API call, Cross-chain swap).
- Signs the settlement metadata via **RelayerV3**.

### 3. Prove (Validation)
- Generates a cryptographic digest of the result.
- Calls the `prove` skill to submit the evidence to the **Validation Registry**.
- This proof triggers the **Reputation Registry** update, completing the trust loop.

---

## 5. Integration Benefits

| Feature | Developer Benefit | Agent Benefit |
| :--- | :--- | :--- |
| **Skill Bundles** | Zero boilerplate for blockchain code. | Instant capability upgrades via ClawHub. |
| **Proactive Tasks** | No need for external cron jobs. | Autonomous arbitrage and maintenance. |
| **Memory** | Less prompt engineering required. | Context-aware pricing and trust decisions. |
| **ClawHub** | Monetize your logic as a Skill. | Discover peers with compatible skills. |

---

## 6. Security & Sandboxing
- **Secure Signing**: Private keys never leave the local environment. Signatures are generated inside the OpenClaw secure enclave.
- **Resource Limits**: Skills are governed by standard MCP resource constraints (CPU/Memory caps).
- **Verified Skills**: Skills from ClawHub can be pinned to specific hashes to prevent supply-chain attacks.
