# MPP Payment Method: `multiversx` (Draft 01)

This document specifies the `multiversx` payment method for the Machine Payments Protocol (MPP), enabling agentic payments over the MultiversX blockchain.

## 1. Abstract

The `multiversx` method leverages the native efficiency of the MultiversX Sharded Protocol. It supports standard EGLD/ESDT transfers, Relayed V3 transactions for gasless operations, and NativeAuth for secure request binding.

## 2. Challenge Parameters

When a Facilitator issues a `402 Payment Required` challenge using the `multiversx` method, it MUST include:

- `address`: The Facilitator's primary receiver wallet (Bech32).
- `tokens`: A comma-separated list of token identifiers (e.g., `EGLD`, `USDC-c76f1f`).
- `amounts`: A comma-separated list of values in Atomic units.
- `chainId`: `1` (Mainnet), `D` (Devnet), or `T` (Testnet).

## 3. The Data Payload Tagging Strategy

To achieve idempotency without complex smart contracts, the agent MUST append the challenge ID to the transaction data field.

### Format: `mpp:<challenge_id>`

**Example**:
If the challenge is `ch_123456`, the transaction `data` field MUST be: `mpp:ch_123456`.

## 4. Payment Header Construction

The AI agent submits the payment proof in the `Authorization` header:

### Charge Intent
```http
Authorization: mpp method="multiversx", tx_hash="<tx_hash>", challenge="<challenge_id>"
```

### Session Intent
```http
Authorization: mpp method="multiversx", session_id="<session_id>", sig="<signature>"
```

## 5. Advanced Features

### 5.1. Relayed V3 (Gasless)
Agents without EGLD can sign a transaction and pass it to the Facilitator via the `x-mpp-relayed-tx` header. The Facilitator co-signs and broadcasts it.

### 5.2. NativeAuth Binding
To prevent replay attacks, the Agent SHOULD include a `NativeAuth` token in the request. The Facilitator verifies that the address who paid the `tx_hash` matches the address in the `NativeAuth` token.

## 6. Token Standards
- **EGLD**: Native gas and currency.
- **ESDT**: Standard digital tokens. The Facilitator MUST support `MultiESDTNFTTransfer` if multiple tokens are requested/paid.
