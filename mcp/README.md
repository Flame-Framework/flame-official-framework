# MCP Servers

MCP servers used by FLAME. These are standalone repositories — clone and build them separately, then point `~/.claude.json` to the built artifacts.

## anypoint-mcp-server

MuleSoft Anypoint Platform Exchange integration. Provides 11 tools for searching, downloading, and publishing API assets.

### Setup

Clone and build the server in a directory of your choice (a sibling of FLAME is a common layout):

```bash
git clone https://github.com/Flame-Framework/anypoint-mcp-server.git
cd anypoint-mcp-server
npm install
npm run build
```

### Configuration

Add to `~/.claude.json`, pointing `args` to the built `dist/index.js` of your local clone:

```json
{
  "mcpServers": {
    "anypoint": {
      "command": "node",
      "args": ["<PATH_TO_ANYPOINT_MCP_SERVER>/dist/index.js"],
      "env": {
        "ANYPOINT_CLIENT_ID": "<your-client-id>",
        "ANYPOINT_CLIENT_SECRET": "<your-client-secret>"
      }
    }
  }
}
```
