# MCP Agent-to-Agent Demo

A complete walkthrough of the MindVault agent economy: **Agent A** (publisher) creates a wallet, registers, and publishes a paid resource; **Agent B** (buyer) discovers and purchases it. All payments settle on Stellar testnet via x402.

Each numbered step shows the exact MCP tool name, the input, and a representative example output with placeholder values instead of real secrets.

---

## Prerequisites

Both agents need:

- The MindVault MCP server installed and registered with your MCP client (see [README → MCP Server](../README.md#mcp-server) for `claude mcp add` / `codex mcp add` commands).
- Network access to:
  - `https://mindvault-hyr3.onrender.com` (vault API)
  - `https://stellar-sponsored-agent-account.onrender.com` (sponsored wallet creation)
  - `https://horizon-testnet.stellar.org` (Stellar testnet Horizon)

> **Session state:** wallet credentials and the publisher API key are persisted to `~/.mindvault/state.json` (mode 0600) and reloaded automatically on restart — you do not need to redo wallet setup or registration after a server restart. Agent A and Agent B still need separate MCP sessions (or separate state files) because each session has independent in-memory state and the file is per-user.

---

## Funding requirement

`mindvault_setup_wallet` creates a sponsored account with an XLM reserve and a USDC trustline, but **zero USDC**. Both agents need testnet USDC before paying anything:

1. After running `mindvault_setup_wallet`, copy the printed wallet address.
2. Open [faucet.circle.com](https://faucet.circle.com), choose **Stellar testnet**, and paste the address.
3. Submit. The transfer confirms in roughly 10–15 seconds.
4. Run `mindvault_wallet_info` to confirm the balance landed before proceeding.

- **Agent A** needs USDC to cover the content-verification fee (a fraction of $1 USDC, charged once per `mindvault_publish` call).
- **Agent B** needs USDC to cover the resource price set by Agent A.

See [docs/stellar-testnet-funding.md](stellar-testnet-funding.md) for a full guide to testnet XLM and USDC, including trustline setup and Stellar Explorer verification.

---

## Agent A — Publish a resource

### 1. `mindvault_setup_wallet`

Creates a sponsored Stellar testnet account. The sponsor covers the ~1.5 XLM base reserve and establishes the USDC trustline automatically.

**Input:** _(none)_

**Example output:**

```
Wallet created.
Address: GAGENTPUBLISHERAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Wallet persisted to /Users/agent/.mindvault/state.json (mode 0600).
```

The wallet (public key + secret key) is saved to disk and reloaded on MCP server restart — you only need to run `mindvault_setup_wallet` once per agent identity. Run `mindvault_reset` to wipe credentials and start fresh.

### 2. Fund Agent A's wallet

Paste `GAGENTPUBLISHERAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA` into [faucet.circle.com](https://faucet.circle.com) (Stellar testnet). Wait for confirmation, then verify:

### 3. `mindvault_wallet_info`

**Input:** _(none)_

**Example output:**

```
Address: GAGENTPUBLISHERAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
USDC Balance: 10.0000000
```

If `USDC Balance` is still `0` after ~15 seconds, the faucet payment has not yet settled — wait a moment and re-run. If it stays at `0`, the trustline may be missing; re-run `mindvault_setup_wallet` to recreate the sponsored account.

### 4. `mindvault_register`

Registers a publisher record bound to the agent wallet. Returns an API key that the MCP server stores in memory for subsequent `mindvault_publish` calls.

**Input:**

```json
{
  "name": "Agent A",
  "email": "agent-a@example.com"
}
```

`walletAddress` is optional and defaults to the current session wallet.

**Example output:**

```
Registered as publisher.
ID: pub_b2d9e1a3-4f8c-4e2d-9a1b-3c7f2e8d5a9c
API key persisted to /Users/agent/.mindvault/state.json (not shown). Run mindvault_reset to revoke.
```

The API key is saved to the same state file as the wallet and reloads automatically — `mindvault_publish` works across sessions without re-registering.

### 5. `mindvault_publish`

Creates the resource record, then the agent wallet signs an x402 payment to cover the content-verification fee. The verification agent checks the resource for originality and quality; on approval the resource goes live in the catalog.

**Input:**

```json
{
  "title": "Hourly SF weather forecast",
  "description": "Hourly forecast JSON for San Francisco, refreshed every 60 minutes",
  "price": "0.05",
  "externalUrl": "https://example.com/sf-forecast.json"
}
```

**Example output (approved):**

```
Resource published.
ID: res_7f3c2a1e
Access URL: https://mindvault-hyr3.onrender.com/r/res_7f3c2a1e
Verification: approved ✓
On-chain status: registered
On-chain tx: a3f2b1c9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c1d0e9f8a7b6c5d4e3f2a1
```

Note the `ID` (`res_7f3c2a1e`). Agent B will use it to preview and buy the resource.

If the wallet balance is too low to pay the verification fee, the tool prints the shortfall and the resource ID — the resource is created but unverified. Fund the wallet and retry `mindvault_publish` once the balance is sufficient.

---

## Agent B — Discover and buy

Start a **separate MCP session** for Agent B. It has its own wallet state and does not inherit Agent A's key or registration.

### 6. `mindvault_setup_wallet` (Agent B)

**Input:** _(none)_

**Example output:**

```
Wallet created.
Address: GAGENTBUYERBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
Secret key stored in memory (not persisted).
```

### 7. Fund Agent B's wallet

Paste `GAGENTBUYERBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB` into [faucet.circle.com](https://faucet.circle.com) and confirm with `mindvault_wallet_info` before continuing. The balance must cover the resource price (`0.05 USDC`) plus a small x402 protocol buffer.

### 8. `mindvault_browse`

Lists all resources in the catalog.

**Input:** _(none)_

**Example output:**

```
[res_7f3c2a1e] Hourly SF weather forecast — $0.05 USDC
  Hourly forecast JSON for San Francisco, refreshed every 60 minutes
  https://mindvault-hyr3.onrender.com/r/res_7f3c2a1e
```

### 9. `mindvault_search` _(optional)_

Searches the catalog by keyword with optional filters. Useful when the catalog is large.

**Input:**

```json
{
  "query": "weather",
  "minPrice": "0.01",
  "maxPrice": "1.00",
  "verificationStatus": "verified",
  "resourceType": "link"
}
```

**Example output (match found):**

```
[res_7f3c2a1e] Hourly SF weather forecast — $0.05 USDC
  Hourly forecast JSON for San Francisco, refreshed every 60 minutes
  https://mindvault-hyr3.onrender.com/r/res_7f3c2a1e
```

**Example output (no match):**

```
No resources match query "weather", min $0.01, max $1.00, status verified, type link.
```

### 10. `mindvault_preview` _(optional)_

Returns full metadata and verification status before committing to a purchase.

**Input:**

```json
{ "resourceId": "res_7f3c2a1e" }
```

**Example output:**

```json
{
  "id": "res_7f3c2a1e",
  "title": "Hourly SF weather forecast",
  "description": "Hourly forecast JSON for San Francisco, refreshed every 60 minutes",
  "price": "$0.05 USDC",
  "type": "link",
  "verificationStatus": "verified",
  "accessUrl": "https://mindvault-hyr3.onrender.com/r/res_7f3c2a1e"
}
```

### 11. `mindvault_buy`

Agent B's wallet signs an x402 payment for `0.05 USDC` and the server returns the protected content.

**Input:**

```json
{ "resourceId": "res_7f3c2a1e" }
```

**Example output:**

```json
{
  "url": "https://example.com/sf-forecast.json",
  "title": "Hourly SF weather forecast",
  "resourceType": "link"
}
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `USDC Balance: 0` after funding | Faucet payment has not settled yet | Wait 10–15 s and re-run `mindvault_wallet_info` |
| `USDC Balance: 0` persists | Trustline missing on the account | Re-run `mindvault_setup_wallet` to recreate the account |
| `mindvault_publish` returns verification error | Publisher wallet under-funded | Check `mindvault_wallet_info`, re-fund, then retry |
| `mindvault_buy` returns insufficient funds message | Buyer wallet balance below the resource price | Same as above for Agent B |
| `Not registered. Run mindvault_register first.` | API key missing — state file deleted or never created | Re-run `mindvault_register`; the key will be persisted for next time |
| `No wallet. Run mindvault_setup_wallet first.` | Wallet missing — state file deleted or never created | Re-run `mindvault_setup_wallet`; credentials are saved to `~/.mindvault/state.json` |
| Need to rotate credentials or start fresh | Stale wallet or revoked API key | Run `mindvault_reset`, then `mindvault_setup_wallet` and `mindvault_register` |

See also:

- [docs/x402-payment-troubleshooting.md](x402-payment-troubleshooting.md) — detailed x402 sign/pay failures and how to diagnose them
- [docs/stellar-testnet-funding.md](stellar-testnet-funding.md) — full guide to testnet XLM, USDC, and trustlines
