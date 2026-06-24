# Glossary

## x402

The x402 protocol standardizes the HTTP 402 Payment Required status code for machine-readable paywalls. A server returns a 402 response with a `PAYMENT-REQUIRED` header encoding the price, destination wallet, network, and payment scheme. The client reads this header, signs a payment, and retries the request with proof of payment attached. MindVault uses `@x402/express` on the server and `@x402/stellar` for client signing.

See also: [Soroban](#soroban), [Sponsored Accounts](#sponsored-accounts)

## Soroban

Soroban is Stellar's smart contract platform. MindVault uses it for the [vault-registry](#vault-registry) contract and for USDC payments via the Stellar Asset Contract (SAC). When a client pays for a resource, it signs a Soroban authorization entry authorizing a USDC transfer from the client's wallet to the creator's wallet.

## vault-registry

The vault-registry is a [Soroban](#soroban) smart contract that maintains an on-chain ledger of published resources. Each entry maps a resource ID to its creator wallet address and price. Resources must be registered here after verification before they appear as on-chain entries in the catalog.

## MCP

MCP (Model Context Protocol) is an open standard for exposing tools to AI agents. MindVault's MCP server lets Claude Code, Codex, and other MCP-enabled agents browse the catalog, pay for resources, and publish new ones — all through natural language without accounts or OAuth.

See the [MCP Server section of the README](../README.md#mcp-server) for installation instructions and available tools.

## Sponsored Accounts

A sponsored account is a Stellar account whose base reserve (~1.5 XLM) and USDC trustline are funded by a sponsoring account rather than the new account holder. MindVault uses the [stellar-sponsored-agent-account](https://github.com/oceans404/stellar-sponsored-agent-account) service so AI agents can receive a Stellar wallet at zero upfront cost and immediately hold USDC.

See also: [MCP](#mcp)
