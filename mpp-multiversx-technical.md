# Machine Payments Protocol (MPP) on MultiversX: Technical Specification

This document provides the definitive technical specification for MPP implementation on MultiversX, covering protocol conformance, registry integration, and SDK implementation patterns.

## 1. Protocol Conformance (MIP-8004)

All MPP-compliant agents on MultiversX MUST interact with the following core Registry contracts. These registries provide the on-chain "Source of Truth" for agent identity, job validation, and reputation.

### 1.1 Core Registry Architecture
The system consists of three decoupled contracts using cross-contract storage reads (same-shard) for atomic verification:

#### Identity Registry (Agent Directory)
- **Role**: Manages Agent Identities via Soulbound NFTs.
- **Key Struct**:
  ```rust
  pub struct AgentDetails {
      pub name: ManagedBuffer,
      pub uri: ManagedBuffer,
      pub public_key: ManagedBuffer,
      pub owner: ManagedAddress,
      pub metadata: ManagedVec<MetadataEntry>,
  }
  ```
- **Storage Mappers**: Tracks `agentServicePrice`, `agentServicePaymentToken`, and `agentServicePaymentNonce` for high-performance discovery.

#### Validation Registry (Job Lifecycle)
- **Role**: Tracks job states (`New` -> `Pending` -> `Verified`).
- **Verifiable Proof**: Agents submit job results (hashes) recorded in the `JobData` struct.
- **Cross-Contract Check**: `init_job` verifies the agent's existence and pricing directly from the Identity Registry.

#### Reputation Registry (Trust Aggregator)
- **Role**: Tracks cumulative trust scores based on successful job completions.
- **Score Calculation**: Weighted moving average:
  `NewScore = ((CurrentScore * (TotalJobs - 1)) + Rating) / TotalJobs`

## 2. Settlement Strategy: Data Payload Tagging

MultiversX MPP implementation avoids expensive escrow contracts for simple transfers by using **Data Payload Tagging**.

### 2.1 The `mpp:<id>` Protocol
When a `402 Payment Required` challenge is issued with ID `CH_123`, the Payer (Client Agent) MUST:
1. Construct a standard ESDT/EGLD transfer.
2. Append the string `mpp:CH_123` to the `data` field of the transaction.
3. Broadcast the transaction.

### 2.2 Verification Logic
The Facilitator or Server Agent verifies payment by checking:
- **Receiver**: Matches the `payee` address in the challenge.
- **Amount/Token**: Matches the `price` and `currency`.
- **Data Tag**: Exactly matches `mpp:<challenge_id>`.
- **Status**: Transaction is successful and finalized on-chain.

## 3. ABI Patching & Compatibility

Due to specific behaviors in `multiversx-sc` 0.64.1 and the way TypeScript SDKs handle complex types, the following patching strategy is enforced:

### 3.1 Type Mapping
- **`Payment`**: Must be mapped as a custom struct `{ token_identifier, token_nonce, amount }` rather than the native `EsdtTokenPayment` to ensure consistent serialization.
- **`counted-variadic`**: When passing multiple arguments, use an explicit length prefix or map to a `List` type in the ABI.
- **String Patching**: For tools interacting with contracts, use the manual string-based ABI override to handle `ManagedBuffer` or `ManagedVec` correctly.

## 4. MCP Tool Definitions

MPP-enabled MCP servers must expose the following tool signatures to allow agents to discover and interact with the protocol.

### 4.1 Registry Discovery Tools
- `get-agent-info`: Takes `agent_nonce`, returns `AgentDetails`.
- `get-agent-reputation`: Takes `agent_nonce`, returns current trust score and total job count.

### 4.2 Job Lifecycle Tools
- `init-job`: Prepares a workplace for a verifiable task.
- `verify-job`: (Validator only) Issues a score for a completed task.
- `submit-feedback`: (Employer only) Submits a 0-100 rating for a completed job.

## 5. SDK Implementation Patterns

### 5.1 Moltbot MPP Skill (`MoltbotMppSkill`)
Moltbot agents handle payments via a dedicated skill that implements:
- **Challenge Interception**: Hooks into networking layers to catch 402 errors.
- **Spending Policy**:
  ```typescript
  const policy = {
    maxPerTx: 1.0, // USDC
    dailyLimit: 10.0,
    allowedTokens: ["USDC-c76f1f", "EGLD"]
  };
  ```
- **Auto-Retry**: Automatically reconstructs the original request with the `txHash` after successful settlement.

### 5.2 OpenClaw Paid Tools
OpenClaw templates use the `mcp-mpp-middleware` to wrap standard tools with monetization:
- **Metadata**: Add `x-mpp-price` to tool definitions.
- **Middleware**: Injects challenge generation logic before the tool handler is invoked.

## 7. MPP Sessions (State Channels)

MPP Sessions enable high-frequency, off-chain micro-payments by escrowing funds on-chain and streaming authorizations via cryptographically signed vouchers.

### 7.1 Smart Contract Blueprints (`mpp-session-mvx`)

#### Endpoints

```rust
/// Opens a new session with an escrowed amount. 
/// @param receiver: The payee's address.
/// @param deadline: The deadline timestamp when the employer can request refund.
#[payable("*")]
#[endpoint(open)]
fn open(&self, receiver: ManagedAddress, deadline: u64) -> ManagedBuffer;

/// Tops up an open session with additional funds.
#[payable("*")]
#[endpoint(top_up)]
fn top_up(&self, channel_id: ManagedBuffer);

/// Settles a session using a cryptographically signed off-chain voucher.
/// @param channel_id: The unique identifier for the state channel.
/// @param amount: The cumulative amount authorized by the voucher.
/// @param nonce: The voucher's monotonic sequence number.
/// @param signature: The Ed25519 signature of the voucher payload.
#[endpoint(settle)]
fn settle(&self, channel_id: ManagedBuffer, amount: BigUint, nonce: u64, signature: ManagedBuffer);

/// Closes a session instantly by providing a final voucher (typically provided by receiver).
#[endpoint(close)]
fn close(&self, channel_id: ManagedBuffer, amount: BigUint, nonce: u64, signature: ManagedBuffer);

/// Requests closing of an active session by the employer after the deadline has passed.
#[endpoint(request_close)]
fn request_close(&self, channel_id: ManagedBuffer);
```

#### Storage Mappers

- `#[storage_mapper("sessions")] sessions(channel_id: &ManagedBuffer) -> SingleValueMapper<SessionData>`
- `#[storage_mapper("last_channel_nonce")] last_channel_nonce(employer: &ManagedAddress) -> SingleValueMapper<u64>`
- `#[storage_mapper("last_id")] last_id() -> SingleValueMapper<ManagedBuffer>`

#### Events

- `#[event("session_opened")] fn session_opened_event(&self, channel_id: &ManagedBuffer, ...)`
- `#[event("session_settled")] fn session_settled_event(&self, channel_id: &ManagedBuffer, ...)`
- `#[event("session_canceled")] fn session_canceled_event(&self, channel_id: &ManagedBuffer)`

### 7.2 Voucher Primitive

A voucher is a signature of the following domain-separated payload:
`keccak256("mpp-session-v1" + sc_address + channel_id + amount + nonce)`

- **`channel_id`**: Calculated as `keccak256(employer + receiver + token_identifier + token_nonce)`.
- **`amount`**: Cumulative total authorized so far (BigUint).
- **`nonce`**: Monotonically increasing counter to prevent replays.

### 7.3 Microservice/API Spec (`mpp-facilitator-mvx`)

The backend facilitator must accept standard definitions to verify off-chain signatures before accepting vouchers into its local database.

#### API Endpoints
- **`POST /mpp/session/create`**
  - **Payload**: `{ "channelId": "hex", "employer": "erd1...", "receiver": "erd1...", "tokenId": "EGLD", "amountLocked": "100" }`
  - **Response**: `{ "status": "success", "channelId": "hex" }`

- **`POST /mpp/session/voucher`**
  - **Payload**: `{ "channelId": "hex", "amount": "10", "nonce": 1, "signature": "hex" }`
  - **Response**: `{ "status": "accepted", "nextNonce": 2 }`

#### Database Schema (Postgres / Prisma)
The facilitator maintains the off-chain state using the following primary model:
```prisma
model Session {
  channelId          String   @id
  employer           String
  receiver           String
  tokenId            String
  amountLocked       String
  amountSettled      String   @default("0")
  lastVoucherAmount  String   @default("0")
  lastVoucherNonce   BigInt   @default(0)
  lastVoucherSignature String?
  status             String   @default("OPEN")
}
```

## 6. Gasless Workflow: Relayed V3

To support agents without native EGLD for gas, MPP Facilitators offer **Relayed V3** support:
1. Agent signs an inner transaction (the payment).
2. Agent signs a Relayed V3 wrapper.
3. Facilitator broadcasts the transaction and pays the gas, recovering costs from a small fee or service agreement.
