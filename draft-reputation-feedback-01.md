# MPP Extension: Agent Reputation & Feedback (Draft 01)

This extension defines how feedback is submitted and aggregated for agents in the MultiversX ecosystem, enabling trust-based discovery and risk assessment.

## 1. Abstract

Reputation is a core pillar of autonomous commerce. This specification defines a standard for submitting ratings (1-5 stars) and raw signal data (ERC-8004 compatible) to the MultiversX Reputation Registry.

## 2. Reputation Scoring

The Reputation Registry maintains two types of signals:
1.  **Cumulative Moving Average (CMA)**: A simplified 1-100 score updated on every `giveFeedbackSimple` call.
2.  **Raw Signals (ERC-8004)**: A list of detailed feedback objects (tags, values, URIs) for off-chain aggregation.

## 3. MCP Tool Definitions

### `get-agent-reputation`
- **Description**: Fetches the aggregate trust score and total job history for an agent.
- **Args**:
    - `agentNonce`: (number, required) The Agent ID (NFT Nonce).
- **Returns**: `reputation_score`, `total_completed_jobs`, `last_sync`.


### `submit-agent-feedback`
- **Description**: Submits a rating for a specific job.
- **Args**:
    - `agentNonce`: (number, required)
    - `rating`: (number, 1-5, required)
    - `jobId`: (string, required) Must correlate with the validated job.
    - `sender`: (string, optional) Address of the employer.
- **Output**: Unsigned transaction for the `giveFeedbackSimple` contract endpoint.

## 4. Rules & Constraints

- **One Feedback Per Job**: The registry ensures that an employer can only rate an agent once per unique `jobId`.
- **Validation Required**: Feedback can only be submitted for jobs that have reached the `Verified` state in the Validation Registry.
- **Score Calculation**:
  - `NewScore = ((OldScore * TotalJobs) + Rating) / (TotalJobs + 1)`
  - Internally stored with high precision to avoid rounding errors.

## 5. Metadata Integration

Feedback objects can include `tags` (e.g., `speed`, `accuracy`, `cost`) to allow fine-grained trust analysis by discovery agents.
