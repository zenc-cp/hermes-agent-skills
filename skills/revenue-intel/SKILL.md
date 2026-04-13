# Revenue Intelligence — Earnings Optimization

## Overview
Revenue intelligence tools for Zenda's $3,000/month survival target. Score content drafts, analyze earnings trends, and detect missed opportunities.

## score_draft
Evaluate a Scribe draft for predicted engagement based on X/Twitter algorithm patterns.
```bash
DRAFT_PATH="${draft_path}"
CLAWDIR=$HOME/claw

python3 -c "
import json, os, re

draft_path = os.path.expanduser('$DRAFT_PATH')

# Read draft
try:
    with open(draft_path) as f:
        content = f.read()
except:
    print(json.dumps({'error': f'Draft not found: $DRAFT_PATH'}))
    raise SystemExit(0)

# Extract thread content (after ## Content or ## Thread Content)
thread = content
for marker in ['## Content', '## Thread Content']:
    idx = content.find(marker)
    if idx >= 0:
        thread = content[idx:]
        break

lines = thread.strip().splitlines()
tweets = [l.strip() for l in thread.split('---') if l.strip()]
if not tweets:
    tweets = [thread]

score = 0
factors = []

# 1. Thread length (3-5 tweets is optimal)
tweet_count = len(tweets)
if 3 <= tweet_count <= 5:
    score += 20
    factors.append({'factor': 'thread_length', 'score': 20, 'detail': f'{tweet_count} tweets (optimal 3-5)'})
elif tweet_count < 3:
    score += 10
    factors.append({'factor': 'thread_length', 'score': 10, 'detail': f'{tweet_count} tweets (too short)'})
else:
    score += 12
    factors.append({'factor': 'thread_length', 'score': 12, 'detail': f'{tweet_count} tweets (long thread)'})

# 2. Tweet length compliance (each < 280 chars)
compliant = sum(1 for t in tweets if len(t) <= 280)
length_score = int((compliant / max(len(tweets), 1)) * 15)
score += length_score
factors.append({'factor': 'length_compliance', 'score': length_score, 'detail': f'{compliant}/{len(tweets)} under 280 chars'})

# 3. Actionability (contains actionable language)
action_words = ['try', 'use', 'check', 'run', 'install', 'configure', 'deploy', 'avoid', 'never', 'always', 'tip:', 'pro tip', 'here\\'s how', 'step']
action_count = sum(1 for w in action_words if w.lower() in thread.lower())
action_score = min(15, action_count * 5)
score += action_score
factors.append({'factor': 'actionability', 'score': action_score, 'detail': f'{action_count} action cues found'})

# 4. Technical depth (code blocks, CVE refs, tool names)
tech_patterns = [r'CVE-\d{4}-\d+', r'\x60[^\x60]+\x60', r'https?://\S+', r'nmap|nuclei|subfinder|burp|ffuf|sqlmap']
tech_count = sum(len(re.findall(p, thread)) for p in tech_patterns)
tech_score = min(15, tech_count * 3)
score += tech_score
factors.append({'factor': 'technical_depth', 'score': tech_score, 'detail': f'{tech_count} technical references'})

# 5. Topic relevance (AI security, cloud, agent, MCP, bug bounty)
hot_topics = ['ai security', 'mcp', 'agent', 'llm', 'cloud', 'zero-day', '0day', 'bug bounty', 'rag', 'supply chain', 'prompt injection']
topic_hits = sum(1 for t in hot_topics if t.lower() in thread.lower())
topic_score = min(20, topic_hits * 5)
score += topic_score
factors.append({'factor': 'topic_relevance', 'score': topic_score, 'detail': f'{topic_hits} hot topics matched'})

# 6. Freshness — check if too similar to recent drafts
drafts_dir = os.path.join(os.path.expanduser('$CLAWDIR'), 'conductor', 'agents', 'scribe', 'drafts')
similar_count = 0
try:
    for fname in sorted(os.listdir(drafts_dir))[-5:]:
        if fname == os.path.basename('$DRAFT_PATH'):
            continue
        with open(os.path.join(drafts_dir, fname)) as f:
            old = f.read().lower()
        # Simple similarity: shared unique words
        new_words = set(thread.lower().split())
        old_words = set(old.split())
        overlap = len(new_words & old_words) / max(len(new_words | old_words), 1)
        if overlap > 0.5:
            similar_count += 1
except: pass

fresh_score = 15 - (similar_count * 5)
fresh_score = max(0, fresh_score)
score += fresh_score
factors.append({'factor': 'freshness', 'score': fresh_score, 'detail': f'{similar_count} similar recent drafts'})

# Final
score = min(100, score)
verdict = 'excellent' if score >= 80 else 'good' if score >= 60 else 'mediocre' if score >= 40 else 'weak'

print(json.dumps({
    'draft': os.path.basename('$DRAFT_PATH'),
    'score': score,
    'verdict': verdict,
    'factors': factors,
    'recommendation': 'Approve for posting' if score >= 60 else 'Consider revising before posting'
}, indent=2))
" 2>/dev/null
```

## revenue_analysis
Query earnings, compute trends, identify top revenue sources.
```bash
PERIOD="${period:-month}"
CLAWDIR=$HOME/claw
DBPATH=$CLAWDIR/claw.db

python3 -c "
import sqlite3, json, os
from datetime import datetime, timezone

db = os.path.expanduser('$DBPATH')
period = '$PERIOD'
days = {'today': 0, 'week': 7, 'month': 30, 'quarter': 90, 'all': 9999}.get(period, 30)

result = {
    'period': period,
    'target': 3000,
    'earnings': {'total': 0, 'by_source': [], 'by_week': []},
    'trading': {'pnl': 0, 'count': 0},
    'bounties': {'active': 0, 'paid': 0},
    'velocity': {},
    'gap_to_target': 3000
}

try:
    conn = sqlite3.connect(db, timeout=2)
    c = conn.cursor()
    tables = [r[0] for r in c.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()]

    if 'earnings' in tables:
        if days == 0:
            where = \"date(created_at)=date('now')\"
        else:
            where = f\"created_at >= datetime('now','-{days} days')\"

        # Total
        c.execute(f'SELECT COALESCE(SUM(amount),0) FROM earnings WHERE {where}')
        result['earnings']['total'] = round(c.fetchone()[0], 2)

        # By source
        c.execute(f'SELECT source, SUM(amount), COUNT(*) FROM earnings WHERE {where} GROUP BY source ORDER BY SUM(amount) DESC')
        for row in c.fetchall():
            result['earnings']['by_source'].append({'source': row[0], 'amount': round(row[1], 2), 'events': row[2]})

        # Monthly total for target comparison
        c.execute(\"SELECT COALESCE(SUM(amount),0) FROM earnings WHERE created_at >= datetime('now','-30 days')\")
        month_total = c.fetchone()[0]
        result['gap_to_target'] = round(3000 - month_total, 2)

        # Weekly velocity: this week vs last week
        c.execute(\"SELECT COALESCE(SUM(amount),0) FROM earnings WHERE created_at >= datetime('now','-7 days')\")
        this_week = c.fetchone()[0]
        c.execute(\"SELECT COALESCE(SUM(amount),0) FROM earnings WHERE created_at >= datetime('now','-14 days') AND created_at < datetime('now','-7 days')\")
        last_week = c.fetchone()[0]
        result['velocity'] = {
            'this_week': round(this_week, 2),
            'last_week': round(last_week, 2),
            'change_pct': round(((this_week - last_week) / max(last_week, 0.01)) * 100, 1)
        }

    if 'trades' in tables:
        c.execute(f\"SELECT COALESCE(SUM(pnl),0), COUNT(*) FROM trades WHERE created_at >= datetime('now','-{days} days')\")
        row = c.fetchone()
        result['trading'] = {'pnl': round(row[0], 2), 'count': row[1]}

    if 'bounties' in tables:
        c.execute(\"SELECT COUNT(*) FROM bounties WHERE status='active'\")
        result['bounties']['active'] = c.fetchone()[0]
        c.execute(f\"SELECT COALESCE(SUM(amount),0) FROM bounties WHERE status='paid' AND created_at >= datetime('now','-{days} days')\")
        result['bounties']['paid'] = round(c.fetchone()[0], 2)

    # Paywall revenue
    rev_file = os.path.expanduser('~/claw/logs/paywall-revenue.jsonl')
    try:
        total = 0.0
        with open(rev_file) as f:
            for line in f:
                try:
                    d = json.loads(line.strip())
                    total += float(d.get('amount', 0))
                except: pass
        result['paywall_usdc'] = round(total, 2)
    except: pass

    conn.close()
except Exception as e:
    result['error'] = str(e)

# Survival status
month_rev = result['earnings']['total'] if period == 'month' else 0
if period == 'month':
    if month_rev >= 3000:
        result['survival_status'] = 'PROFITABLE'
    elif month_rev >= 2000:
        result['survival_status'] = 'BREAKING EVEN'
    else:
        result['survival_status'] = 'BURNING CASH'

print(json.dumps(result, indent=2))
" 2>/dev/null
```

## opportunity_scan
Check for unapproved proposals, unsubmitted findings, or pending revenue actions.
```bash
CLAWDIR=$HOME/claw

python3 -c "
import json, os, glob

clawdir = os.path.expanduser('$CLAWDIR')
result = {'opportunities': [], 'total_estimated_value': 0}

# 1. Check pending evolution proposals (unapproved radar proposals)
pending_evo = f'{clawdir}/workspace/ops/data/pending-evolution.json'
try:
    with open(pending_evo) as f:
        proposals = json.load(f).get('proposals', [])
    pending = [p for p in proposals if p.get('status') == 'pending']
    if pending:
        result['opportunities'].append({
            'type': 'pending_evolution',
            'count': len(pending),
            'detail': [{'id': p['id'], 'reason': p['reason'][:100]} for p in pending],
            'action': 'Review and approve/reject pending evolution proposals'
        })
except: pass

# 2. Check pending scribe drafts (unapproved content)
drafts_dir = f'{clawdir}/conductor/agents/scribe/drafts'
try:
    pending_drafts = []
    for fname in os.listdir(drafts_dir):
        if not fname.endswith('.md'):
            continue
        fpath = os.path.join(drafts_dir, fname)
        with open(fpath) as f:
            header = f.read(500)
        if 'PENDING_REVIEW' in header:
            pending_drafts.append(fname)
    if pending_drafts:
        result['opportunities'].append({
            'type': 'pending_drafts',
            'count': len(pending_drafts),
            'detail': pending_drafts[:5],
            'action': 'Review and approve content drafts for posting',
            'estimated_value': 0  # Brand building, hard to quantify
        })
except: pass

# 3. Check pending actions directory
pending_dir = f'{clawdir}/data/pending-actions'
try:
    pending_actions = []
    if os.path.isdir(pending_dir):
        for fname in os.listdir(pending_dir):
            if fname.endswith('.json'):
                with open(os.path.join(pending_dir, fname)) as f:
                    pending_actions.append(json.load(f))
    if pending_actions:
        result['opportunities'].append({
            'type': 'pending_actions',
            'count': len(pending_actions),
            'detail': [{'type': a.get('type'), 'created': a.get('created')} for a in pending_actions],
            'action': 'Review and execute pending actions'
        })
except: pass

# 4. Check scan queue (queued but not started)
scan_queue = f'{clawdir}/data/scan-queue.json'
try:
    with open(scan_queue) as f:
        q = json.load(f)
    queued = [s for s in q.get('queue', []) if not any(p.get('scan_id') == s.get('scan_id') for p in q.get('processed', []))]
    if queued:
        result['opportunities'].append({
            'type': 'queued_scans',
            'count': len(queued),
            'detail': [{'domain': s.get('domain'), 'type': s.get('scan_type')} for s in queued[:5]],
            'action': 'Run queued security scans — potential consulting revenue',
            'estimated_value': len(queued) * 500
        })
        result['total_estimated_value'] += len(queued) * 500
except: pass

# 5. Check unseen bounty programs (hunter has new matches)
try:
    with open(f'{clawdir}/conductor/agents/hunter/state.json') as f:
        hunter_state = json.load(f)
    new_jobs = hunter_state.get('last_new_total', 0)
    if new_jobs > 0:
        result['opportunities'].append({
            'type': 'new_bounty_programs',
            'count': new_jobs,
            'action': 'Review new bounty matches from last Hunter cycle',
            'estimated_value': new_jobs * 200  # Rough per-bounty estimate
        })
        result['total_estimated_value'] += new_jobs * 200
except: pass

# 6. Check ops pending patch
pending_patch = f'{clawdir}/workspace/ops/data/pending-patch.json'
try:
    with open(pending_patch) as f:
        patch = json.load(f)
    if patch.get('status') == 'pending_approval':
        result['opportunities'].append({
            'type': 'pending_patch',
            'detail': {'file': patch.get('file', ''), 'instruction': patch.get('instruction', '')[:100]},
            'action': 'Approve or reject pending code patch'
        })
except: pass

# 7. Check trader signals (actionable but not executed)
try:
    with open(f'{clawdir}/conductor/agents/trader/signals.json') as f:
        signals = json.load(f)
    actionable = [s for s in (signals if isinstance(signals, list) else signals.get('signals', [])) if s.get('action') in ('BUY', 'SELL')]
    if actionable:
        result['opportunities'].append({
            'type': 'trader_signals',
            'count': len(actionable),
            'detail': [{'action': s.get('action'), 'asset': s.get('asset', 'ETH')} for s in actionable[:3]],
            'action': 'Review actionable trading signals'
        })
except: pass

print(json.dumps(result, indent=2))
" 2>/dev/null
```

## Trigger Phrases
- "score this draft" / "rate the draft" → score_draft
- "how are earnings?" / "revenue report" / "earnings this month" → revenue_analysis
- "any opportunities?" / "what am I missing?" / "pending actions" → opportunity_scan
- "survival status" → revenue_analysis (period=month)
