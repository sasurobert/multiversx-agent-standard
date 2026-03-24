# MPP Extension: Payment Discovery & Priced Tools (Draft 01)

This extension defines how AI agents discover payment requirements and service pricing using OpenAPI extensions and standard manifest patterns on MultiversX.

## 1. Abstract

For an autonomous economy to scale, agents must be able to "read the price tag" before initiating a job. This specification leverages the `x-payment-info` OpenAPI extension and the Agent Registration File (ARF) to expose pricing.

## 2. OpenAPI Extensions

Facilitators and Agents exposing MCP tools SHOULD include the following extensions in their `openapi.json`:

### `x-payment-info`
Specifies the pricing for a particular endpoint or tool.
```json
"x-payment-info": {
  "method": "multiversx",
  "currency": "EGLD",
  "amount": "0.001",
  "intent": "charge",
  "description": "Price: 0.001 EGLD"
}
```

### `x-service-info`
Provides descriptive metadata for the service.
```json
"x-service-info": {
  "category": "computation",
  "sla": "99.9%",
  "verifiable": true
}
```

## 3. Discovery Flow

1.  **Metadata Discovery**: Agent finds the peer via the Identity Registry or `search-agents` tool.
2.  **Manifest Fetch**: Agent calls `get-agent-manifest(nonce)`.
3.  **Pricing Analysis**:
    - MCP server patches the tool descriptions with pricing information retrieved from the manifest.
    - Resulting tool description: `[Paid: 1.00 USDC] Analyzes source code for bugs.`
4.  **Pre-Approval**: The LLM sees the price and can ask the user for approval or use a pre-set budget.

## 4. MultiversX ARF Integration

The ARF (Agent Registration File) stored in TxData MUST contain a `pricing` object if the agent is not free.

```json
{
  "name": "AuditBot",
  "pricing": {
    "type": "x402",
    "token": "EGLD",
    "rate_per_request": "0.01",
    "facilitator": "https://api.facilitator.com"
  }
}
```

## 5. Facilitator Discovery

Facilitators MUST host `/.well-known/agent-card.json` which points to their OpenAPI definition, allowing recursive discovery of all priced services hosted by that facilitator.
