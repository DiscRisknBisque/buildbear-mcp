# BuildBear MCP Server

This MCP server allows supported harnesses and LLMs to use the BuildBear API via tool calls. The documented [Sandbox API](https://www.buildbear.io/docs/api-reference/sandbox-api), [Explorer API](https://www.buildbear.io/docs/api-reference/explorer-api), and [Custom RPC Methods](https://www.buildbear.io/docs/api-reference/custom-rpc-methods) are implemented.

## Contents

- [Prerequisites](#prerequisites): If you need a BuildBear API key
- Installation
  - [Claude Desktop](#claude-desktop)
  - [Codex Desktop](#codex-desktop)
  - [OpenCode](#opencode)
- [End-to-End Sandbox Workflow](#end-to-end-sandbox-workflow)

## Prerequisites

1. Get a BuildBear API key from the [BuildBear dashboard](https://app.buildbear.io).
2. Clone this repository and build the server:

```bash
git clone https://github.com/DiscRisknBisque/buildbear-mcp.git
cd buildbear-mcp
npm install
npm run build
```

The built entrypoint is `build/index.js`. In the configuration examples below, replace `/path/to/buildbear-mcp` with the absolute path to your clone.

## Claude Desktop

1. Open your Claude Desktop MCP config (see the [MCP quickstart](https://modelcontextprotocol.io/quickstart/server#testing-your-server-with-claude-for-desktop) for the file location on your OS).
2. Add a `buildbear` server entry:

```json
{
  "mcpServers": {
    "buildbear": {
      "command": "node",
      "args": ["/path/to/buildbear-mcp/build/index.js"],
      "env": {
        "BB_API_KEY": "your-buildbear-api-key"
      }
    }
  }
}
```

3. Restart Claude Desktop.

## Codex Desktop

Codex stores MCP configuration in `~/.codex/config.toml` (or a project-scoped `.codex/config.toml` in trusted projects). You can configure the server via the UI (**Settings → MCP Servers → Add server**) or by editing the file directly.

### Option A: CLI

```bash
codex mcp add buildbear \
  --env BB_API_KEY=your-buildbear-api-key \
  -- node /path/to/buildbear-mcp/build/index.js
```

Verify the server is registered with `codex mcp list`. In a Codex session, run `/mcp` to confirm the BuildBear tools are available.

### Option B: `config.toml`

```toml
[mcp_servers.buildbear]
command = "node"
args = ["/path/to/buildbear-mcp/build/index.js"]
enabled = true
startup_timeout_sec = 20

[mcp_servers.buildbear.env]
BB_API_KEY = "your-buildbear-api-key"
```

See the [Codex MCP documentation](https://developers.openai.com/codex/mcp) for additional options such as tool timeouts and approval modes.

## OpenCode

Add the server under the `mcp` key in your OpenCode config (`~/.config/opencode/opencode.json`, or `opencode.json` in your project root). See the [OpenCode MCP docs](https://dev.opencode.ai/docs/mcp-servers/) for details.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "buildbear": {
      "type": "local",
      "command": ["node", "/path/to/buildbear-mcp/build/index.js"],
      "enabled": true,
      "environment": {
        "BB_API_KEY": "your-buildbear-api-key"
      }
    }
  }
}
```

You can also add the server interactively with `opencode mcp add`, then confirm it with `opencode mcp list`.

## End-to-End Sandbox Workflow

This example shows how an agent can spin up its own sandbox, fund a wallet, snapshot state, and execute on-chain transactions to simulate economic activity.

### Prompt

Give your agent a prompt like:

> Using the BuildBear MCP tools, set up an economic simulation on a fork of Ethereum mainnet. Create a sandbox, derive a wallet from the sandbox mnemonic, fund that wallet with 10 ETH and 10,000 USDC, snapshot the initial state, then send 1 ETH to `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` to simulate a payment. Report balances before and after.

### Expected Workflow

1. **Create a sandbox** — `create-sandbox` with `chainId: 1` (Ethereum mainnet fork).
2. **Fetch sandbox details** — `fetch-sandbox-details` returns the `sandboxId`, `rpcUrl`, `mnemonic`, and `explorerUrl`. The agent derives an address from the mnemonic (e.g. HD path `m/44'/60'/0'/0/0`).
3. **Fund native tokens** — `native-token-faucet` with the agent's address and a balance of `10000000000000000000` wei (10 ETH).
4. **Fund ERC-20 tokens** — `erc20-token-faucet` with the agent's address, the mainnet USDC contract (`0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`), and a balance of `10000000000` (10,000 USDC, 6 decimals).
5. **Snapshot initial state** — `snapshot` so the agent can roll back and re-run scenarios from the same starting point.
6. **Execute transactions** — send transactions against the sandbox `rpcUrl` using standard Ethereum JSON-RPC. BuildBear sandboxes expose unlocked accounts derived from the sandbox mnemonic, so the agent can sign and broadcast transfers directly.

### Sending a Transaction via RPC

After funding, the agent can transfer ETH with `eth_sendTransaction`:

```bash
curl -X POST "https://rpc.buildbear.io/<sandbox-id>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_sendTransaction",
    "params": [{
      "from": "0xYourAgentAddress",
      "to": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
      "value": "0xDE0B6B3A7640000"
    }],
    "id": 1
  }'
```

The `value` field is `1 ETH` in hex wei (`0xDE0B6B3A7640000`). The agent can check balances with `eth_getBalance` and inspect the transfer in the sandbox explorer URL returned by `fetch-sandbox-details`.

### Why This Works for Agents

- **Self-funded wallets** — faucet tools let the agent mint unlimited native and ERC-20 tokens without manual setup.
- **Isolated environment** — each simulation runs in a private fork; nothing touches a public testnet or mainnet.
- **Repeatable experiments** — snapshots let the agent reset to a known state and compare outcomes across runs.
- **Full composability** — once funded, the agent can interact with any contract already present on the forked chain (DEXes, lending protocols, etc.) using the sandbox RPC and explorer tools (`get-contract-abi`, `get-source-code`).

When the simulation is done, the agent can call `delete-sandbox` to clean up.
