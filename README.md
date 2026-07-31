# macOS Runner MCP

A GitHub Actions workflow that turns a macOS runner into an interactive AI agent-accessible machine via [agent-term](https://github.com/michelledupuis/agent-term) MCP server exposed through a Cloudflare Tunnel.

## What It Does

When triggered, this workflow:

1. Spins up a **macOS runner** (Apple Silicon)
2. Installs **agent-term** (MCP terminal server)
3. Installs **cloudflared** (Cloudflare Tunnel)
4. Starts agent-term and exposes it through a public HTTPS URL
5. Keeps the runner alive for **5 hours** (configurable)

You can then connect any AI agent (Claude Code, VS Code, Cursor, etc.) to the tunnel URL and interact with the macOS runner as if it were a local terminal.

## Quick Start

### 1. Trigger the Workflow

Go to **Actions > macOS Runner MCP > Run workflow** and click "Run workflow".

### 2. Get Connection Info

After the workflow starts, check the **Step Summary** or download the `macos-runner-connection-info` artifact for:

- **MCP Password**: `MacOS`
- **MCP URL**: `https://<random-words>.trycloudflare.com/mcp`
- **Auth Token**: `<generated-token>`

### 3. Connect Your AI Agent

#### Claude Code

```bash
claude mcp add macos-runner --transport http https://<your-url>/mcp \
  --header "Authorization: Bearer <YOUR-TOKEN>"
```

#### VS Code / Cursor

Add to `.vscode/mcp.json` or `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "macos-runner": {
      "url": "https://<your-url>/mcp",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Bearer <YOUR-TOKEN>"
      }
    }
  }
}
```

### 4. Use It

Once connected, your AI agent can:

```bash
# Run commands
start_shell_session → send_shell_input "ls -la\r" → read_shell_output

# Read TUI output (vim, htop, etc.)
read_shell_output with mode: "screen"

# Search the screen
search_screen with pattern: "error"
```

## How It Works

```
AI Agent ←→ Cloudflare Tunnel ←→ macOS Runner
                                    ├── agent-term (MCP server, port 8808)
                                    │   └── PTY sessions (bash, zsh, etc.)
                                    └── Full macOS environment
                                        ├── Xcode, iOS Simulator
                                        ├── Android SDK + Emulator
                                        ├── Homebrew, Rust, Python, Node
                                        └── All standard macOS tools
```

## Security

- **Bearer token auth**: Every request requires `Authorization: Bearer <token>`
- **Token generated per run**: A unique 64-char hex token is generated each time
- **Default bind**: agent-term binds to `127.0.0.1` (not exposed directly)
- **Cloudflare Tunnel**: Only the tunnel endpoint is public; no ports are opened
- **Auto-cleanup**: Sessions idle for 30 minutes are terminated; runner shuts down after keep-alive period

## Configuration

### Keep-Alive Duration

Set via workflow dispatch input (default: 5 hours, max: 5):

```
Actions > macOS Runner MCP > Run workflow → keep_alive_hours: 3
```

### Custom Token

The token is auto-generated. To use a custom token, edit the workflow and replace the token generation step.

## Prerequisites

- This repository must have **Actions enabled**
- The workflow uses the default `GITHUB_TOKEN` (no additional secrets needed)
- agent-term is downloaded from [michelledupuis/agent-term](https://github.com/michelledupuis/agent-term) releases

## Troubleshooting

### Tunnel URL not appearing

- Wait up to 2 minutes for cloudflared to establish the tunnel
- Check the `cf.log` artifact for errors
- Verify agent-term is running: check `agent-term.log`

### Agent can't connect

- Verify the token matches what's in the artifact
- Check that the tunnel URL is still valid (tunnels expire after ~24 hours of inactivity)
- Ensure your AI agent supports streamable-http transport

### Runner timed out

- The default timeout is 5.5 hours (5h keep-alive + buffer)
- Re-trigger the workflow if you need more time

## License

MIT
