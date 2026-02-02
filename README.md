# MX-8004: MultiversX Agent Standard (Specs)

This repository contains the technical specifications, economic blueprints, and standard drafts for the **MultiversX Agent Economy**.

## Overview
MX-8004 (inspired by ERC-8004) defines the trustless infrastructure for AI Agents on MultiversX, focusing on:
- **Identity Registry**: Dynamic NFTs for agent discovery and communication.
- **Trust Layer**: Decentralized reputation and verifiable proof systems.
- **Payment Rails**: x402 micro-payment standard for native monetization.
- **Context Gateway**: MCP (Model Context Protocol) for LLM-to-Chain interaction.

## Contents
- [MIP-8004 Draft](multiversx_eip_draft.md): The official standard proposal.
- [MX-8004 Smart Contract Specs](mx8004_technical_specs.md): Blueprints for Registry, Reputation, and Validation contracts.
- [x402 Facilitator Specs](x402_facilitator_technical_specs.md): Implementation guide for payment settlement and relaying.
- [MCP Server Specs](mcp_server_technical_specs.md): Technical details for the MultiversX Model Context Protocol server.
- [Ecosystem Implementation Plan](ecosystem_implementation_plan.md): The 4-phase roadmap for launching the economy.
- [Agentic Use Cases](AGENTIC_USECASES.md): Real-world applications of the standard.

## Architecture
The system leverages MultiversX's unique features like:
- **TxData Persistence**: High-speed manifest storage without state bloat.
- **Relayed V3**: Gasless experiences for users and agents.
- **x402**: Native 402 "Payment Required" flows.

---
© 2026 MultiversX Agent Working Group.
