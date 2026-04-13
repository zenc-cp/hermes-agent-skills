# Agent Tools — Direct Control of All 6 Zenda Agents

## Overview
Directly trigger any Zenda agent via their exec.sh scripts. Each tool sets the appropriate environment variables, runs the agent with a timeout, and captures output.

## guard_scan
Run Guard security scan on a target domain.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/guard/exec.sh
TARGET="${target:-}"
TRIGGER_ID="zc-guard-$(date +%s)"

# Check current state
STATE=$(python3 -c "
import json
try:
    with open('$CONDUCTORDIR/agents/guard/state.json') as f: d = json.load(f)
    print(d.get('status','unknown'))
except: print('unknown')
" 2>/dev/null)

if [ "$STATE" = "running" ]; then
  echo "{\"status\": \"skipped\", \"reason\": \"guard is already running\"}"
  exit 0
fi

export TRIGGER_ACTION="scan"
export TRIGGER_TARGET="$TARGET"
export TRIGGER_ID="$TRIGGER_ID"
export TRIGGER_SOURCE="zeroclaw"

OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"guard\", \"action\": \"scan\", \"target\": \"$TARGET\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json; print(json.dumps('''$OUTPUT'''[:2000]))" 2>/dev/null || echo '""')}"
```

## guard_health
Run Guard service health check.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/guard/exec.sh
TRIGGER_ID="zc-guard-$(date +%s)"

export TRIGGER_ACTION="health"
export TRIGGER_TARGET=""
export TRIGGER_ID="$TRIGGER_ID"
export TRIGGER_SOURCE="zeroclaw"

OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"guard\", \"action\": \"health\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:2000]))" <<< "$OUTPUT" 2>/dev/null || echo '""')}"
```

## trader_run
Run Trader full cycle — fetch signals, evaluate, paper trade.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/trader/exec.sh

# Check current state
STATE=$(python3 -c "
import json
try:
    with open('$CONDUCTORDIR/agents/trader/state.json') as f: d = json.load(f)
    print(d.get('status','unknown'))
except: print('unknown')
" 2>/dev/null)

if [ "$STATE" = "running" ]; then
  echo "{\"status\": \"skipped\", \"reason\": \"trader is already running\"}"
  exit 0
fi

# Trader uses legacy mode (no TRIGGER_ACTION needed for full cycle)
OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"trader\", \"action\": \"run\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:2000]))" <<< "$OUTPUT" 2>/dev/null || echo '""')}"
```

## hunter_poll
Run Hunter bounty polling — checks Toku, Superteam, ClawTasks, Immunefi, Bountycaster.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/hunter/exec.sh

STATE=$(python3 -c "
import json
try:
    with open('$CONDUCTORDIR/agents/hunter/state.json') as f: d = json.load(f)
    print(d.get('status','unknown'))
except: print('unknown')
" 2>/dev/null)

if [ "$STATE" = "running" ]; then
  echo "{\"status\": \"skipped\", \"reason\": \"hunter is already running\"}"
  exit 0
fi

TRIGGER_ID="zc-hunter-$(date +%s)"
export TRIGGER_ACTION="hunt"
export TRIGGER_TARGET=""
export TRIGGER_ID="$TRIGGER_ID"
export TRIGGER_SOURCE="zeroclaw"

OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"hunter\", \"action\": \"poll\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:2000]))" <<< "$OUTPUT" 2>/dev/null || echo '""')}"
```

## scribe_draft
Run Scribe content draft generation.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/scribe/exec.sh
PLATFORM="${platform:-farcaster}"
TOPIC="${topic:-security}"
TRIGGER_ID="zc-scribe-$(date +%s)"

STATE=$(python3 -c "
import json
try:
    with open('$CONDUCTORDIR/agents/scribe/state.json') as f: d = json.load(f)
    print(d.get('status','unknown'))
except: print('unknown')
" 2>/dev/null)

if [ "$STATE" = "running" ]; then
  echo "{\"status\": \"skipped\", \"reason\": \"scribe is already running\"}"
  exit 0
fi

export TRIGGER_ACTION="draft"
export TRIGGER_TARGET="${PLATFORM}:${TOPIC}"
export TRIGGER_ID="$TRIGGER_ID"
export TRIGGER_SOURCE="zeroclaw"

OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"scribe\", \"action\": \"draft\", \"platform\": \"$PLATFORM\", \"topic\": \"$TOPIC\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:2000]))" <<< "$OUTPUT" 2>/dev/null || echo '""')}"
```

## scout_discover
Run Scout OSINT discovery — CVEs, trending repos, CISA KEVs.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/scout/exec.sh

STATE=$(python3 -c "
import json
try:
    with open('$CONDUCTORDIR/agents/scout/state.json') as f: d = json.load(f)
    print(d.get('status','unknown'))
except: print('unknown')
" 2>/dev/null)

if [ "$STATE" = "running" ]; then
  echo "{\"status\": \"skipped\", \"reason\": \"scout is already running\"}"
  exit 0
fi

# Scout uses legacy mode for full discovery cycle
OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"scout\", \"action\": \"discover\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:2000]))" <<< "$OUTPUT" 2>/dev/null || echo '""')}"
```

## ops_health
Run Ops health status check on all services.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/ops/exec.sh
TRIGGER_ID="zc-ops-$(date +%s)"

export TRIGGER_ACTION="status"
export TRIGGER_TARGET=""
export TRIGGER_ID="$TRIGGER_ID"
export TRIGGER_SOURCE="zeroclaw"

OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"ops\", \"action\": \"health\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:2000]))" <<< "$OUTPUT" 2>/dev/null || echo '""')}"
```

## ops_audit
Run Ops weekly audit — disk, memory, stale processes, log rotation.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/ops/exec.sh
TRIGGER_ID="zc-ops-$(date +%s)"

export TRIGGER_ACTION="audit"
export TRIGGER_TARGET=""
export TRIGGER_ID="$TRIGGER_ID"
export TRIGGER_SOURCE="zeroclaw"

OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?

# Also read the audit report if generated
REPORT=""
if [ -f "$HOME/claw/workspace/ops/data/audit-report.json" ]; then
  REPORT=$(cat "$HOME/claw/workspace/ops/data/audit-report.json" 2>/dev/null | head -c 3000)
fi

echo "{\"agent\": \"ops\", \"action\": \"audit\", \"exit_code\": $EXIT, \"report\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:3000]))" <<< "$REPORT" 2>/dev/null || echo '""')}"
```

## ops_brain
Run Ops brain reasoning cycle — system-wide assessment.
```bash
CONDUCTORDIR=$HOME/claw/conductor
EXEC=$CONDUCTORDIR/agents/ops/exec.sh
TRIGGER_ID="zc-ops-$(date +%s)"

export TRIGGER_ACTION="brain"
export TRIGGER_TARGET=""
export TRIGGER_ID="$TRIGGER_ID"
export TRIGGER_SOURCE="zeroclaw"

OUTPUT=$(timeout 120 bash "$EXEC" 2>&1 | tail -20)
EXIT=$?
echo "{\"agent\": \"ops\", \"action\": \"brain\", \"exit_code\": $EXIT, \"output\": $(python3 -c "import json,sys; print(json.dumps(sys.stdin.read()[:2000]))" <<< "$OUTPUT" 2>/dev/null || echo '""')}"
```

## Trigger Phrases
- "run guard scan on example.com" → guard_scan
- "check services" → guard_health or ops_health
- "run trader" → trader_run
- "check bounties" / "run hunter" → hunter_poll
- "draft a post about X" → scribe_draft
- "run scout" / "check CVEs" → scout_discover
- "ops health" → ops_health
- "run audit" → ops_audit
- "run brain" → ops_brain
