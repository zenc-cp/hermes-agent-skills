# Evolution — Self-Improvement Pipeline

## Overview
The evolution skill enables ZeroClaw to propose, review, and apply modifications to its own skills and agent configurations. Every change follows the PLAN→BUILD→VALIDATE→MONITOR pipeline and requires Zen's approval.

## propose_skill_change
Read a current skill file, propose a modification, and save to pending-evolution.json.
```bash
SKILL_PATH="${skill_path}"
REASON="${reason}"
PROPOSED="${proposed_content}"
CLAWDIR=$HOME/claw
PENDING=$CLAWDIR/workspace/ops/data/pending-evolution.json
PROPOSAL_ID="evo-$(date +%s)-$$"

mkdir -p "$(dirname "$PENDING")"

python3 -c "
import json, os, hashlib
from datetime import datetime, timezone

pending_file = os.path.expanduser('$PENDING')
skill_path = os.path.expanduser('$SKILL_PATH')

# Load existing proposals
proposals = []
try:
    with open(pending_file) as f:
        proposals = json.load(f).get('proposals', [])
except: pass

# Check max 3 pending
active = [p for p in proposals if p.get('status') == 'pending']
if len(active) >= 3:
    print(json.dumps({'error': 'Max 3 pending proposals. Approve or expire existing ones first.'}))
    raise SystemExit(0)

# Read current content
current_content = ''
try:
    with open(skill_path) as f:
        current_content = f.read()
except:
    current_content = '[FILE NOT FOUND]'

# Create proposal
proposal = {
    'id': '$PROPOSAL_ID',
    'skill_path': '$SKILL_PATH',
    'reason': '$REASON',
    'current_content_hash': hashlib.sha256(current_content.encode()).hexdigest()[:16],
    'current_content': current_content,
    'proposed_content': '''$PROPOSED''',
    'status': 'pending',
    'created': datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ'),
    'validated': False,
    'applied': False
}

proposals.append(proposal)

with open(pending_file, 'w') as f:
    json.dump({'proposals': proposals}, f, indent=2)

print(json.dumps({
    'status': 'proposed',
    'id': '$PROPOSAL_ID',
    'skill_path': '$SKILL_PATH',
    'reason': '$REASON',
    'current_lines': len(current_content.splitlines()),
    'proposed_lines': len('''$PROPOSED'''.splitlines())
}))
" 2>/dev/null
```

## review_skill
Compare current vs proposed content, validate syntax.
```bash
PROPOSAL_ID="${proposal_id}"
CLAWDIR=$HOME/claw
PENDING=$CLAWDIR/workspace/ops/data/pending-evolution.json

python3 -c "
import json, os, subprocess, tempfile

pending_file = os.path.expanduser('$PENDING')
proposal_id = '$PROPOSAL_ID'

try:
    with open(pending_file) as f:
        data = json.load(f)
except:
    print(json.dumps({'error': 'No pending proposals file found'}))
    raise SystemExit(0)

# Find proposal
proposal = None
for p in data.get('proposals', []):
    if p.get('id') == proposal_id:
        proposal = p
        break

if not proposal:
    print(json.dumps({'error': f'Proposal {proposal_id} not found'}))
    raise SystemExit(0)

result = {
    'id': proposal_id,
    'skill_path': proposal['skill_path'],
    'reason': proposal['reason'],
    'status': proposal['status'],
    'validation': {}
}

# Read current file
current = ''
try:
    with open(os.path.expanduser(proposal['skill_path'])) as f:
        current = f.read()
except:
    current = proposal.get('current_content', '')

proposed = proposal.get('proposed_content', '')

# Generate diff summary
current_lines = current.splitlines()
proposed_lines = proposed.splitlines()
result['diff_summary'] = {
    'current_lines': len(current_lines),
    'proposed_lines': len(proposed_lines),
    'lines_added': len(proposed_lines) - len(current_lines)
}

# Validate syntax based on file extension
path = proposal['skill_path']
valid = True
errors = []

if path.endswith('.sh'):
    with tempfile.NamedTemporaryFile(mode='w', suffix='.sh', delete=False) as tmp:
        tmp.write(proposed)
        tmp_path = tmp.name
    r = subprocess.run(['bash', '-n', tmp_path], capture_output=True, text=True)
    os.unlink(tmp_path)
    if r.returncode != 0:
        valid = False
        errors.append(f'bash syntax: {r.stderr.strip()}')

elif path.endswith('.py'):
    with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as tmp:
        tmp.write(proposed)
        tmp_path = tmp.name
    r = subprocess.run(['python3', '-m', 'py_compile', tmp_path], capture_output=True, text=True)
    os.unlink(tmp_path)
    if r.returncode != 0:
        valid = False
        errors.append(f'python syntax: {r.stderr.strip()}')

elif path.endswith('.toml'):
    try:
        import tomllib
        tomllib.loads(proposed)
    except ImportError:
        try:
            import tomli
            tomli.loads(proposed)
        except ImportError:
            pass  # No toml parser available, skip
    except Exception as e:
        valid = False
        errors.append(f'toml syntax: {str(e)}')

elif path.endswith('.js'):
    with tempfile.NamedTemporaryFile(mode='w', suffix='.js', delete=False) as tmp:
        tmp.write(proposed)
        tmp_path = tmp.name
    r = subprocess.run(['node', '-c', tmp_path], capture_output=True, text=True)
    os.unlink(tmp_path)
    if r.returncode != 0:
        valid = False
        errors.append(f'js syntax: {r.stderr.strip()}')

result['validation'] = {'valid': valid, 'errors': errors}

# Update proposal validation status
proposal['validated'] = valid
with open(pending_file, 'w') as f:
    json.dump(data, f, indent=2)

print(json.dumps(result, indent=2))
" 2>/dev/null
```

## apply_evolution
Apply an approved proposal to a git staging branch (NOT main).
```bash
PROPOSAL_ID="${proposal_id}"
CLAWDIR=$HOME/claw
PENDING=$CLAWDIR/workspace/ops/data/pending-evolution.json
REPODIR=$HOME/workspace/claw-stack-jp

python3 -c "
import json, os, subprocess
from datetime import datetime, timezone

pending_file = os.path.expanduser('$PENDING')
repo = os.path.expanduser('$REPODIR')
proposal_id = '$PROPOSAL_ID'

try:
    with open(pending_file) as f:
        data = json.load(f)
except:
    print(json.dumps({'error': 'No pending proposals'}))
    raise SystemExit(0)

proposal = None
for p in data.get('proposals', []):
    if p.get('id') == proposal_id:
        proposal = p
        break

if not proposal:
    print(json.dumps({'error': f'Proposal {proposal_id} not found'}))
    raise SystemExit(0)

if not proposal.get('validated'):
    print(json.dumps({'error': 'Proposal not validated. Run review_skill first.'}))
    raise SystemExit(0)

if proposal.get('status') != 'pending':
    print(json.dumps({'error': f'Proposal status is {proposal[\"status\"]}, expected pending'}))
    raise SystemExit(0)

# Create staging branch
branch = f'evolution/{proposal_id}'
try:
    subprocess.run(['git', '-C', repo, 'checkout', '-b', branch], capture_output=True, text=True, check=True)
except:
    # Branch might exist, try switching
    subprocess.run(['git', '-C', repo, 'checkout', branch], capture_output=True, text=True)

# Write the proposed content
target = os.path.join(repo, proposal['skill_path'].replace('~/claw/', '').replace(os.path.expanduser('~/claw/'), ''))
os.makedirs(os.path.dirname(target), exist_ok=True)
with open(target, 'w') as f:
    f.write(proposal['proposed_content'])

# Commit
subprocess.run(['git', '-C', repo, 'add', target], capture_output=True, text=True)
msg = f'evolution: {proposal[\"reason\"][:80]}'
subprocess.run(['git', '-C', repo, 'commit', '-m', msg], capture_output=True, text=True)

# Switch back to main
subprocess.run(['git', '-C', repo, 'checkout', 'main'], capture_output=True, text=True)

# Update proposal
proposal['status'] = 'staged'
proposal['branch'] = branch
proposal['applied_at'] = datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ')
proposal['applied'] = True
with open(pending_file, 'w') as f:
    json.dump(data, f, indent=2)

print(json.dumps({
    'status': 'staged',
    'id': proposal_id,
    'branch': branch,
    'skill_path': proposal['skill_path'],
    'reason': proposal['reason'],
    'message': f'Staged on branch {branch}. Zen must approve before merging to main.'
}))
" 2>/dev/null
```

## list_learnings
Read knowledge ledger, agent memories, and skill scores to identify improvement opportunities.
```bash
CLAWDIR=$HOME/claw

python3 -c "
import json, os, glob

clawdir = os.path.expanduser('$CLAWDIR')
result = {'skill_scores': {}, 'agent_memories': {}, 'brain_insights': [], 'gate_decisions': [], 'pending_evolutions': []}

# Skill scores
try:
    with open(f'{clawdir}/data/learning/skill_scores.json') as f:
        result['skill_scores'] = json.load(f)
except: pass

# Agent memories (from agent-memory system)
for agent in ['guard', 'trader', 'hunter', 'scribe', 'scout', 'ops']:
    mem_file = f'{clawdir}/workspace/{agent}/data/memory.json'
    try:
        with open(mem_file) as f:
            mem = json.load(f)
            result['agent_memories'][agent] = {
                'entries': len(mem) if isinstance(mem, list) else len(mem.get('memories', [])),
                'summary': str(mem)[:200]
            }
    except:
        result['agent_memories'][agent] = {'entries': 0}

# Brain insights — last 5
brain_log = f'{clawdir}/workspace/ops/data/brain-log.jsonl'
try:
    with open(brain_log) as f:
        lines = [l.strip() for l in f.readlines() if l.strip()]
    for line in lines[-5:]:
        try: result['brain_insights'].append(json.loads(line))
        except: pass
except: pass

# Gate decisions — last 5
gate_log = f'{clawdir}/data/learn/gate-decisions.jsonl'
try:
    with open(gate_log) as f:
        lines = [l.strip() for l in f.readlines() if l.strip()]
    for line in lines[-5:]:
        try: result['gate_decisions'].append(json.loads(line))
        except: pass
except: pass

# Pending evolutions
pending_file = f'{clawdir}/workspace/ops/data/pending-evolution.json'
try:
    with open(pending_file) as f:
        proposals = json.load(f).get('proposals', [])
    for p in proposals:
        if p.get('status') == 'pending':
            result['pending_evolutions'].append({
                'id': p['id'],
                'skill_path': p['skill_path'],
                'reason': p['reason'],
                'created': p.get('created'),
                'validated': p.get('validated', False)
            })
except: pass

# Learning feedback — last 5
feedback_log = f'{clawdir}/data/learning/feedback.jsonl'
try:
    with open(feedback_log) as f:
        lines = [l.strip() for l in f.readlines() if l.strip()]
    result['recent_feedback'] = []
    for line in lines[-5:]:
        try: result['recent_feedback'].append(json.loads(line))
        except: pass
except: pass

print(json.dumps(result, indent=2))
" 2>/dev/null
```

## Trigger Phrases
- "propose a change to X" / "improve X skill" → propose_skill_change
- "review proposal X" / "validate evolution" → review_skill
- "apply evolution X" / "stage the change" → apply_evolution
- "what can we improve?" / "list learnings" → list_learnings

## Evolution Pipeline
```
PLAN → BUILD → VALIDATE → MONITOR
  │       │        │          │
  │       │        │          └─ Check outcomes in next brain cycle
  │       │        └─ bash -n / py_compile / toml parse
  │       └─ Write proposed content to pending-evolution.json
  └─ Identify need from brain-log, skill scores, gate decisions
```
