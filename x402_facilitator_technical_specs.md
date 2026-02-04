# Technical Specification: x402 Facilitator for MultiversX

This specification defines the architecture, API, and implementation details for the **x402 Facilitator** on the MultiversX blockchain. The facilitator acts as a bridge between buyers (agents/users) and sellers (resource servers), providing shared infrastructure for payment verification and settlement.

---

## 1. Role and Responsibilities

The x402 Facilitator is an off-chain service that:
1.  **Verifies** x402 payment payloads against specific requirements.
2.  **Settles** payments on-chain, optionally acting as a **Relayer** (Relayed V3) to enable gasless experiences for users.
3.  **Discovery**: Provides a registry of supported networks, tokens, and merchants.

---

## 2. API Blueprint

### 2.1. Verification API

**Endpoint**: `POST /verify`
**Description**: Validates a signed payment payload without broadcasting it to the blockchain.

#### Payload (JSON)
```json
{
  "scheme": "exact",
  "payload": {
    "nonce": "uint64",
    "value": "string (BigInt)",
    "receiver": "string (Bech32)",
    "sender": "string (Bech32)",
    "gasPrice": "uint64",
    "gasLimit": "uint64",
    "data": "string (optional)",
    "chainID": "string",
    "version": 1,
    "options": 0,
    "signature": "hex-string",
    "validAfter": "uint64 (optional)",
    "validBefore": "uint64 (optional)"
  },
  "requirements": {
    "payTo": "string (Bech32)",
    "amount": "string (BigInt)",
    "asset": "string (Ticker/Identifier)",
    "network": "string (CAIP-like, e.g., multiversx:D)",
    "extra": {
        "assetTransferMethod": "direct | esdt"
    }
  }
}
```

#### Logic Execution (Facilitator Side)
1.  **Static Checks**: Verify `validAfter` and `validBefore` against current Unix time.
2.  **Signature Verification**: Use `ed25519` to verify that the `payload` was signed by the `sender`.
3.  **Requirements Match**: 
    - Verify `receiver == payTo`.
    - Verify `value >= amount` (for EGLD) OR parse `data` for `MultiESDTNFTTransfer` to verify token and amount.
4.  **Simulation**: Call the MultiversX API `/transaction/simulate` to ensure the transaction will succeed on-chain (checks balances, nonce, SC logic).

#### Response
```json
{
  "isValid": true,
  "payer": "string (sender bech32)"
}
```

---

### 2.2. Settlement API

**Endpoint**: `POST /settle`
**Description**: Fully broadcasts the verified payment payload. If the facilitator is a relayer, it attaches its own signature.

#### Behavior
1.  **Verification**: Automatically calls the logic of `/verify` first.
2.  **Relaying**: 
    - If the transaction is **Relayed V3**: The facilitator sets itself as `relayer`, adds `relayerSignature`, and sets `version: 2`.
3.  **Broadcast**: Submits to the network via `/transaction/send`.
4.  **Finalization**: Polls the transaction status until it is `successful` or `failed`.

#### Response
```json
{
  "success": true,
  "transaction": "tx-hash-hex",
  "network": "multiversx:D",
  "payer": "bech32-address"
}
```

---

## 3. Storage & Infrastructure Blueprints

### 3.1. Database (Idempotency)
Used to prevent replay attacks and track settlement history.
- **Table**: `Settlements`
- **Columns**:
    - `id` (Primary Key)
    - `tx_hash` (Unique Index)
    - `sender`
    - `receiver`
    - `amount`
    - `asset`
    - `status` (Pending, Success, Failed)
    - `created_at`

### 3.2. Facilitator Wallet (The Relayer)
- **Role**: Must hold a balance of EGLD to pay for gas if acting as a relayer.
- **Multi-Shard Architecture**: To support users across all shards, the facilitator must manage multiple wallets (one per shard).
    - **Implementation**: A directory of PEM files (e.g., `shard0.pem`, `shard1.pem`) loaded at startup.
    - **Logic**: When relaying, the service identifies the sender's shard and signs with the corresponding relayer wallet.
- **Security**: Private keys MUST be stored in a Hardware Security Module (HSM) or a secure Vault (e.g., HashiCorp Vault), or loaded from secure PEM files in a restricted environment.

---

## 4. Token Support (MultiversX Specifics)

### 4.1. EGLD (Native)
- **Transfer Method**: `direct`.
- **Validation**: Check `value` and `receiver` fields in the payload.

### 4.2. ESDT (Token)
- **Transfer Method**: `esdt`.
- **Validation**: 
    - Transaction `receiver` must be the `sender` (self-send to trigger the move).
    - `data` field must start with `MultiESDTNFTTransfer`.
    - Arguments in `data` (hex-encoded) must match the requested `asset`, `amount`, and `payTo`.

---

## 5. Summary Table (Facilitator Responsibilities)

| **Interoperability** | Uniform API for all x402-compliant servers |

---

## 6. Safety and Security Analysis

The x402 Facilitator is designed as a **Trustless Proxy**. It cannot steal funds because it does not hold user private keys.

### 6.1. Why is it safe?
- **Non-Custodial Payloads**: The buyer (user/agent) signs the payment payload locally. The facilitator only receives a signed message. It cannot alter the `receiver` or `amount` without invalidating the signature.
- **On-Chain Enforcement**: Even if a facilitator is malicious, it can only broadcast what the user signed. The MultiversX protocol itself enforces that the signer had the funds and authorized the move.
- **Simulation Layer**: By simulating every transaction before settlement, the facilitator acts as a "Guardian", catching errors (like low balance or wrong nonce) before gas is wasted.
- **Idempotency**: The facilitator database ensures that a single payment payload cannot be settled twice, protecting the buyer from double-spending the same intent.

---

## 7. Value Proposition

### 7.1. Benefits for Users (The Buyers)
- **Gasless Experience**: Via Relayed V3, users don't need to hold EGLD for gas. They can pay strictly in USDC or other tokens.
- **Instant Satisfaction**: The 402 "Challenge-Response" loop happens in milliseconds. No complex subscription management needed.
- **Streaming Compatibility**: Works perfectly for micro-payments as small as $0.001, enabling use cases like "Pay-per-Prompt".

### 7.2. Benefits for Agents (The Sellers)
- **Zero Infrastructure**: Agent developers don't need to run their own blockchain nodes. They simply point their `paymentMiddleware` to the facilitator.
- **Automated Settlement**: Agents receive funds directly to their wallets. The facilitator handles the complex logic of monitoring transaction finality.
- **Discovery & SEO**: Being listed on a trusted facilitator's `/list` endpoint provides instant visibility to the ecosystem of x402-compatible buyers.
