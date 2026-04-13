# Zenda Manager — System Awareness & Control

## Overview
The central manager skill for ZeroClaw. Provides live system awareness across all 6 agents, chain messaging, earnings, brain reasoning, and the ability to trigger agent actions.

## system_status
Get full system status: all 6 agents, services, earnings, chain activity.
```bash
CLAWDIR=$HOME/claw
DBPATH=$CLAWDIR/claw.db
CONDUCTORDIR=$CLAWDIR/conductor

echo "{"

# Services health
echo '"services": {'
FIRST=1
for svc in claw-core wa-proxy caddy claw-paywall conductor-v2 claw-board; do
  [ $FIRST -eq 0 ] && echo ","
  FIRST=0
  if systemctl is-active --quiet "$svc" 2>/dev/null; then
    echo "\"$svc\": \"up\""
  else
    echo "\"$svc\": \"down\""
  fi
done
echo "},"

# Agent states
echo '"agents": {'
FIRST=1
for agent in guard trader hunter scribe scout ops; do
  [ $FIRST -eq 0 ] && echo ","
  FIRST=0
  STATEFILE=$CONDUCTORDIR/agents/$agent/state.json
  if [ -f "$STATEFILE" ]; then
    STATE=$(python3 -c "
import json
with open('$STATEFILE') as f: d = json.load(f)
print(json.dumps({
  'status': d.get('status','unknown'),
  'lastrun': d.get('lastrun','never'),
  'runcount': d.get('runcount',0),
  'lastresult': d.get('lastresult','')
}))
" 2>/dev/null || echo '{"status":"error"}')
    echo "\"$agent\": $STATE"
  else
    echo "\"$agent\": {\"status\":\"no state file\"}"
  fi
done
echo "},"

# Earnings summary
echo '"earnings":'
python3 -c "
import sqlite3, json, os
db = os.path.expanduser('$DBPATH')
result = {'today': 0, 'week': 0, 'month': 0, 'trade_pnl_30d': 0}
try:
    conn = sqlite3.connect(db, timeout=2)
    c = conn.cursor()
    tables = [r[0] for r in c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()]
    if 'earnings' in tables:
        c.execute(\"SELECT COALESCE(SUM(amount),0) FROM earnings WHERE date(created_at)=date('now')\")
        result['today'] = round(c.fetchone()[0], 2)
        c.execute(\"SELECT COALESCE(SUM(amount),0) FROM earnings WHERE created_at >= datetime('now','-7 days')\")
        result['week'] = round(c.fetchone()[0], 2)
        c.execute(\"SELECT COALESCE(SUM(amount),0) FROM earnings WHERE created_at >= datetime('now','-30 days')\")
        result['month'] = round(c.fetchone()[0], 2)
    if 'trades' in tables:
        c.execute(\"SELECT COALESCE(SUM(pnl),0) FROM trades WHERE created_at >= datetime('now','-30 days')\")
        result['trade_pnl_30d'] = round(c.fetchone()[0], 2)
    conn.close()
except: pass
print(json.dumps(result))
" 2>/dev/null || echo '{"today":0,"week":0,"month":0}'
echo ","

# Chain queue stats
echo '"chain_activity":'
python3 -c "
import sqlite3, json, os
db = os.path.expanduser('$DBPATH')
stats = {'total': 0, 'pending': 0, 'processing': 0, 'completed': 0, 'failed': 0, 'dead': 0}
try:
    conn = sqlite3.connect(db, timeout=2)
    c = conn.cursor()
    tables = [r[0] for r in c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()]
    if 'chain_queue' in tables:
        c.execute('SELECT status, COUNT(*) FROM chain_queue GROUP BY status')
        for row in c.fetchall():
            stats[row[0]] = row[1]
            stats['total'] += row[1]
    conn.close()
except: pass
print(json.dumps(stats))
" 2>/dev/null || echo '{"total":0}'
echo ","

# System resources
echo '"system": {'
echo "\"disk_pct\": \"$(df -h / 2>/dev/null | awk 'NR==2{print $5}')\","
echo "\"mem_pct\": \"$(free 2>/dev/null | awk 'NR==2{printf "%.0f%%", $3/$2*100}')\","
echo "\"load\": \"$(cat /proc/loadavg 2>/dev/null | cut -d' ' -f1)\","
echo "\"uptime\": \"$(uptime -p 2>/dev/null)\""
echo "}"

echo "}"
```

## agent_detail
Get detailed status for a specific agent.
```bash
AGENT="${agent}"
CLAWDIR=$HOME/claw
DBPATH=$CLAWDIR/claw.db
CONDUCTORDIR=$CLAWDIR/conductor
STATEFILE=$CONDUCTORDIR/agents/$AGENT/state.json

python3 -c "
import sqlite3, json, os

agent = '$AGENT'
clawdir = os.path.expanduser('$CLAWDIR')
dbpath = os.path.expanduser('$DBPATH')
conductor = os.path.expanduser('$CONDUCTORDIR')

result = {'agent': agent}

# State file
statefile = f'{conductor}/agents/{agent}/state.json'
try:
    with open(statefile) as f:
        result['state'] = json.load(f)
except:
    result['state'] = {'status': 'no state file'}

# Last DB entry
try:
    conn = sqlite3.connect(dbpath, timeout=2)
    c = conn.cursor()
    tables = [r[0] for r in c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()]
    if 'conductorlog' in tables:
        c.execute('SELECT ts, action, result, exitcode FROM conductorlog WHERE agent=? ORDER BY id DESC LIMIT 5', (agent,))
        result['recent_runs'] = [{'ts': r[0], 'action': r[1], 'result': r[2], 'exitcode': r[3]} for r in c.fetchall()]
    # Chain messages for this agent
    if 'chain_queue' in tables:
        c.execute('SELECT id, ts, from_agent, to_agent, action, status FROM chain_queue WHERE to_agent=? OR from_agent=? ORDER BY id DESC LIMIT 10', (agent, agent))
        result['chain_messages'] = [{'id': r[0], 'ts': r[1], 'from': r[2], 'to': r[3], 'action': r[4], 'status': r[5]} for r in c.fetchall()]
    conn.close()
except Exception as e:
    result['db_error'] = str(e)

# Skill health
try:
    health_file = f'{clawdir}/logs/skill-health.json'
    with open(health_file) as f:
        all_health = json.load(f)
    result['skill_health'] = all_health.get(agent, {})
except:
    result['skill_health'] = {}

# Agent-specific data
if agent == 'trader':
    try:
        with open(f'{conductor}/agents/trader/paper-pnl.json') as f:
            result['paper_pnl'] = json.load(f)
    except: pass
    try:
        with open(f'{conductor}/agents/trader/signals.json') as f:
            result['signals'] = json.load(f)
    except: pass
elif agent == 'hunter':
    try:
        with open(f'{conductor}/agents/hunter/seen-jobs.json') as f:
            result['total_seen_jobs'] = len(json.load(f).get('seen', {}))
    except: pass
elif agent == 'scout':
    try:
        with open(f'{conductor}/agents/scout/latest-intel.json') as f:
            intel = json.load(f)
            result['intel_summary'] = {k: len(v) if isinstance(v, list) else v for k, v in intel.items() if k != 'raw'}
    except: pass
elif agent == 'ops':
    try:
        health_hist = f'{clawdir}/workspace/ops/data/health-history.jsonl'
        with open(health_hist) as f:
            lines = f.readlines()
        if lines:
            result['last_health'] = json.loads(lines[-1])
    except: pass
    try:
        with open(f'{clawdir}/workspace/ops/data/audit-report.json') as f:
            result['last_audit'] = json.load(f)
    except: pass

print(json.dumps(result, indent=2))
" 2>/dev/null
```

## earnings_report
Get earnings report for a time period.
```bash
bash $HOME/claw/scripts/earning-report.sh "${period:-week}" 2>/dev/null || python3 -c "
import sqlite3, json, os
db = os.path.expanduser('~/claw/claw.db')
period = '${period:-week}'
days = {'today': 0, 'week': 7, 'month': 30, 'all': 9999}.get(period, 7)
result = {'period': period, 'sources': [], 'total': 0, 'trade_pnl': 0}
try:
    conn = sqlite3.connect(db, timeout=2)
    c = conn.cursor()
    tables = [r[0] for r in c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()]
    if 'earnings' in tables:
        if days == 0:
            where = \"date(created_at)=date('now')\"
        else:
            where = f\"created_at >= datetime('now','-{days} days')\"
        c.execute(f'SELECT source, SUM(amount), COUNT(*) FROM earnings WHERE {where} GROUP BY source ORDER BY SUM(amount) DESC')
        for row in c.fetchall():
            result['sources'].append({'source': row[0], 'amount': round(row[1], 2), 'count': row[2]})
            result['total'] += row[1]
        result['total'] = round(result['total'], 2)
    if 'trades' in tables:
        c.execute(f\"SELECT COALESCE(SUM(pnl),0), COUNT(*) FROM trades WHERE created_at >= datetime('now','-{days} days')\")
        row = c.fetchone()
        result['trade_pnl'] = round(row[0], 2)
        result['trade_count'] = row[1]
    if 'bounties' in tables:
        c.execute(\"SELECT COUNT(*) FROM bounties WHERE status='active'\")
        result['active_bounties'] = c.fetchone()[0]
    conn.close()
except Exception as e:
    result['error'] = str(e)
print(json.dumps(result, indent=2))
" 2>/dev/null
```

## brain_log
Get recent brain decisions and learnings.
```bash
CLAWDIR=$HOME/claw
BRAIN_LOG=$CLAWDIR/workspace/ops/data/brain-log.jsonl
FEEDBACK_LOG=$CLAWDIR/data/learning/feedback.jsonl
SKILL_SCORES=$CLAWDIR/data/learning/skill_scores.json

python3 -c "
import json, os

result = {'brain_decisions': [], 'recent_feedback': [], 'skill_scores': {}}

# Brain log — last 10 entries
brain_log = os.path.expanduser('$BRAIN_LOG')
try:
    with open(brain_log) as f:
        lines = [l.strip() for l in f.readlines() if l.strip()]
    for line in lines[-10:]:
        try: result['brain_decisions'].append(json.loads(line))
        except: pass
except: pass

# Learning feedback — last 10 entries
feedback_log = os.path.expanduser('$FEEDBACK_LOG')
try:
    with open(feedback_log) as f:
        lines = [l.strip() for l in f.readlines() if l.strip()]
    for line in lines[-10:]:
        try: result['recent_feedback'].append(json.loads(line))
        except: pass
except: pass

# Skill scores
scores_file = os.path.expanduser('$SKILL_SCORES')
try:
    with open(scores_file) as f:
        result['skill_scores'] = json.load(f)
except: pass

print(json.dumps(result, indent=2))
" 2>/dev/null
```

## chain_activity
Get recent chain queue activity and gate decisions.
```bash
CLAWDIR=$HOME/claw
DBPATH=$CLAWDIR/claw.db
GATE_LOG=$CLAWDIR/data/learn/gate-decisions.jsonl

python3 -c "
import sqlite3, json, os

result = {'queue_stats': {}, 'recent_messages': [], 'gate_decisions': []}

# Chain queue from SQLite
db = os.path.expanduser('$DBPATH')
try:
    conn = sqlite3.connect(db, timeout=2)
    c = conn.cursor()
    tables = [r[0] for r in c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()]
    if 'chain_queue' in tables:
        # Stats by status
        c.execute('SELECT status, COUNT(*) FROM chain_queue GROUP BY status')
        for row in c.fetchall():
            result['queue_stats'][row[0]] = row[1]
        # Recent 15 messages
        c.execute('SELECT id, ts, from_agent, to_agent, action, status, result FROM chain_queue ORDER BY id DESC LIMIT 15')
        for row in c.fetchall():
            result['recent_messages'].append({
                'id': row[0], 'ts': row[1], 'from': row[2], 'to': row[3],
                'action': row[4], 'status': row[5], 'result': (row[6] or '')[:200]
            })
    conn.close()
except: pass

# Gate decisions — last 10
gate_log = os.path.expanduser('$GATE_LOG')
try:
    with open(gate_log) as f:
        lines = [l.strip() for l in f.readlines() if l.strip()]
    for line in lines[-10:]:
        try: result['gate_decisions'].append(json.loads(line))
        except: pass
except: pass

print(json.dumps(result, indent=2))
" 2>/dev/null
```

## run_agent
Trigger an agent to run a specific action. Creates a trigger file that conductor-v2 picks up.
```bash
AGENT="${agent}"
ACTION="${action}"
CLAWDIR=$HOME/claw
TRIGGERDIR=$CLAWDIR/data/triggers
TRIGGER_ID="zc-$(date +%s)-$$"

mkdir -p "$TRIGGERDIR"

# Validate agent
case "$AGENT" in
  guard|trader|hunter|scribe|scout|ops) ;;
  *) echo "{\"error\": \"Invalid agent: $AGENT. Must be: guard, trader, hunter, scribe, scout, ops\"}"; exit 1 ;;
esac

# Write trigger file for conductor-v2
python3 -c "
import json, os
trigger = {
    'agent': '$AGENT',
    'action': '$ACTION',
    'target': '',
    'id': '$TRIGGER_ID',
    'source': 'zeroclaw-manager',
    'ts': '$(date -u +%FT%TZ)'
}
triggerdir = os.path.expanduser('$TRIGGERDIR')
filepath = os.path.join(triggerdir, f'{trigger[\"id\"]}.json')
with open(filepath, 'w') as f:
    json.dump(trigger, f, indent=2)
print(json.dumps({'status': 'triggered', 'agent': '$AGENT', 'action': '$ACTION', 'trigger_id': '$TRIGGER_ID'}))
" 2>/dev/null
```

## Trigger Phrases
- "how's the system?" / "status" → system_status
- "how's guard doing?" / "check trader" → agent_detail
- "how much did we earn?" / "earnings this week" → earnings_report
- "what has brain decided?" / "brain log" → brain_log
- "chain activity" / "what are agents saying to each other?" → chain_activity
- "run guard scan" / "trigger trader" → run_agent
