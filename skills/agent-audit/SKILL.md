# Agent Audit — AI Agent Security Scanner

## Overview
Scan AI agent codebases for security vulnerabilities using agent-audit.
Covers OWASP Agentic Top 10 (2026): prompt injection, tool misuse, MCP poisoning,
identity/privilege issues, supply chain risks, code execution, memory poisoning.

## scan_agent_project
Scan a directory or GitHub repo for AI agent security issues.
```bash
TARGET="${target_path:-$HOME/claw}"
FORMAT="${format:-json}"
SEVERITY="${severity:-medium}"

# Ensure agent-audit is installed
if ! command -v agent-audit &>/dev/null; then
  pip3 install --user agent-audit 2>/dev/null
fi

# Run scan
agent-audit scan "$TARGET" --severity "$SEVERITY" --format "$FORMAT" 2>/dev/null

echo ""
echo "---"
echo "Scan complete. Format: $FORMAT | Min severity: $SEVERITY"
```

## scan_sarif
Generate SARIF output for GitHub Security tab integration.
```bash
TARGET="${target_path:-$HOME/claw}"
OUTPUT="${output_path:-$HOME/claw/data/agent-audit-results.sarif}"

if ! command -v agent-audit &>/dev/null; then
  pip3 install --user agent-audit 2>/dev/null
fi

agent-audit scan "$TARGET" --format sarif --output "$OUTPUT" 2>/dev/null

echo "SARIF report saved: $OUTPUT"
echo "Upload to GitHub Security tab or attach to AgentArmor report."
```

## self_audit
Audit Zenda's own agent stack for security hygiene.
```bash
CLAWDIR=$HOME/claw

if ! command -v agent-audit &>/dev/null; then
  pip3 install --user agent-audit 2>/dev/null
fi

echo "=== Zenda Self-Audit ==="
echo ""

# Scan conductor agents
echo "## Conductor Agents"
agent-audit scan "$CLAWDIR/conductor" --severity low --format json 2>/dev/null

echo ""
echo "## WA-Proxy"
agent-audit scan "$CLAWDIR/wa-proxy" --severity low --format json 2>/dev/null

echo ""
echo "## ZeroClaw Skills"
agent-audit scan "$CLAWDIR/zeroclaw/workspace/skills" --severity low --format json 2>/dev/null

echo ""
echo "Self-audit complete."
```

## inspect_mcp
Inspect an MCP server for security issues (tool poisoning, shadowing, drift).
```bash
MCP_CMD="${mcp_command}"

if ! command -v agent-audit &>/dev/null; then
  pip3 install --user agent-audit 2>/dev/null
fi

if [ -z "$MCP_CMD" ]; then
  echo '{"error": "Provide mcp_command, e.g.: npx -y @modelcontextprotocol/server-filesystem /tmp"}'
  exit 0
fi

agent-audit inspect stdio -- $MCP_CMD 2>/dev/null
```

## Trigger Phrases
- "audit agent" / "scan for prompt injection" → scan_agent_project
- "self audit" / "audit zenda" / "security hygiene" → self_audit
- "generate sarif" / "github security" → scan_sarif
- "inspect mcp" / "check mcp server" → inspect_mcp
