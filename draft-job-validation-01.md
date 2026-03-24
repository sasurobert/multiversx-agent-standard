# MPP Extension: Verifiable Inference Job Validation (Draft 01)

This extension defines the standard lifecycle for delegating, proving, and verifying computational jobs between agents using the MultiversX Machine Payments Protocol (MPP).

## 1. Abstract

In an agentic economy, payments are often contingent on the successful completion of a task. This specification provides a standard state machine for jobs, enabling cryptographic proof submission and multi-party validation (Oracles or Owners).

## 2. Job Lifecycle

A job in the MX-8004 ecosystem follows these states:

1.  **New**: Job initialized, payment Escrowed (if applicable).
2.  **Pending**: Execution in progress.
3.  **ValidationRequested**: Agent has submitted a result and is awaiting verification.
4.  **Verified**: Job successfully validated, payment released.
5.  **Failed / Disputed**: Validation failed.

## 3. MCP Tool Definitions

The following tools MUST be implemented by a compliant MX-8004 MCP server to support this extension.

### `is-job-verified`
- **Description**: Returns the current status of a job in the Validation Registry.
- **Args**:
    - `jobId`: (string, required) The unique identifier of the job.
- **Returns**: `Verified`, `Pending`, or `NotFound`.

### `submit-job-proof`
- **Description**: Stores evidence of job completion on-chain.
- **Args**:
    - `jobId`: (string, required)
    - `proofHash`: (string, required) Hex-encoded hash of the result.
    - `sender`: (string, optional) Bech32 address of the Agent.
- **Output**: Unsigned transaction for `submit_proof`.

### `validation-request`
- **Description**: (Agent Owner only) Requests validation from a specific validator.
- **Args**:
    - `jobId`: (string, required)
    - `validatorAddress`: (string, required) Bech32 address.
    - `requestUri`: (string, required) URI for validation details.
    - `requestHash`: (string, required) Hash of the request content.
    - `sender`: (string, optional) Bech32 address.
- **Output**: Unsigned transaction for `validation_request`.

### `validation-response`
- **Description**: (Validator only) Submits the validation score.
- **Args**:
    - `requestHash`: (string, required) The hash of the request being responded to.
    - `response`: (number, required, 0-100) Validation score.
    - `responseUri`: (string, required) URI for response details.
    - `responseHash`: (string, required) Hash of the response content.
    - `tag`: (string, required) Classification tag (e.g. "verified").
    - `sender`: (string, optional) Bech32 address.
- **Output**: Unsigned transaction for `validation_response`.

### `verify-job`
- **Description**: Convenience tool to finalize job verification (maps to `validation_response` with score 100).
- **Args**:
    - `jobId`: (string, required) Used as the request hash.
    - `sender`: (string, optional) Bech32 address.
- **Output**: Unsigned transaction using success defaults.

## 4. Smart Contract Integration

Compliant implementations interact with the `ValidationRegistry` contract:

| Function | Purpose |
| :--- | :--- |
| `init_job` | Called by Client to open a job. |
| `submit_proof` | Called by Agent to attach evidence. |
| `validation_request` | Called by Agent/Client to trigger Oracle. |
| `validation_response` | Called by Validator to finalize. |

## 5. Security Requirements

- **Idempotency**: The `jobId` MUST be unique and should correlate with the MPP payment challenge.
- **Authorization**: Only the registered owner of the Agent NFT or the delegated Validator can transition the job to `Verified`.
