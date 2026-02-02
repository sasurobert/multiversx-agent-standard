# Competitive Benchmarking: Agent Ecosystems

## 1. Base (Coinbase AgentKit)
- **Model**: "Managed Infra". Coinbase runs the nodes/relayers.
- **Experience**: "Based Agent" (Replit template).
- **Key Insight**: Users don't manage keys or full nodes. They just write the "Agent Logic" (Python/JS) and use a provided Wallet Provider. **Ease of Use: 10/10.**
- **Payment**: Standard ERC-20 (USDC). No native streaming standard.

## 2. Solana (Eliza / ai16z)
- **Model**: "Framework First". `Eliza` is a TypeScript library (`npm install @eliza/core`).
- **Experience**: Developers clone a repo, config env vars, and run.
- **Key Insight**: Tightly integrated into DeFi/DAO culture. The "Starter Kit" is just a GitHub repo with good docs.
- **Payment**: Tokens ($ELIZA) used for governance/access.

## 3. Olas (Autonolas)
- **Model**: "Full Decentralization". Complex registry of Components, Services, and Agents.
- **Experience**: High learning curve. Requires stacking multiple NFTs (Component -> Service -> Agent).
- **Key Insight**: Powerful but too complex for "One-Click".

---

# The "Moltbot" Winning Formula
To beat these, MultiversX/Moltbot needs to combine **Base's Ease of Use** with **Solana's Framework Power**, plus our unique **x402 Streaming**.

## The "Ultimate Starter Kit" Spec
We don't just provide a standard; we provide a **Hosted ecosystem**.

1.  **The "Molt Launcher" (CLI/Web)**:
    - User types: `npx create-molt-bot my-agent`
    - Logic: Generates a project pre-configured with `mx-sdk`.
    - Identity: Automatically mints the Dynamic NFT (Registry).

2.  **Shared Infrastructure (The "Molt Cloud")**:
    - **Molt MCP Server**: A public API (`mcp.molt.bot`) that gives agents free read-access to chain state. *No RPC node needed for the user.*
    - **Molt Relayer**: An HTTP endpoint (`relay.molt.bot`) where agents POST signed txs. *No gas management needed initially (Relayer pays gas, takes cut).*
    - **x402 Facilitator**: A Websocket (`wss://pay.molt.bot`) that pushes "Payment Events" to agents.

3.  **The "Agent Loop" (Standardized)**:
    ```javascript
    // agent.js
    import { MoltBot } from '@molt/sdk';
    
    const bot = new MoltBot({ key: process.env.PRIVATE_KEY });
    
    bot.onStreamStart((user, rate) => {
        console.log(`Getting paid ${rate} from ${user}`);
        bot.startThinking(user);
    });
    
    bot.onMessage(async (msg) => {
        const reply = await bot.ai.generate(msg);
        bot.reply(reply); // Signs & Relays automatically
    });
    ```
