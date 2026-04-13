---
name: zenops-ops
description: >-
  ZenOps stack operational playbook. Covers VM deploy commands, service
  management, database access, agent debugging, and common fixes. Load when
  operating the ZenOps production system, debugging services, deploying changes,
  or checking system health.
category: zenops
---

# ZenOps Operations Playbook

## 1. Deploy Commands

### Full Deploy (push to main → auto-deploy)
```bash
git add -A && git commit -m "type(scope): description" && git push origin main
# Deploy triggers automatically via .github/workflows/deploy.yml
# Wait ~2 min, then verify:
gh run list --repo zenc-cp/claw-stack-jp --workflow="deploy.yml" --limit 1
```

### Quick Patch (VM-only, no commit needed)
```bash
gh workflow run "Run Remote Command" --repo zenc-cp/claw-stack-jp --ref main \
  -f command='cd ~/claw && git pull origin main && sudo systemctl restart conductor-v2 claw-board'
```

### Deploy + Restart Specific Service
```bash
gh workflow run "Run Remote Command" --repo zenc-cp/claw-stack-jp --ref main \
  -f command='cd ~/claw && git pull origin main && sudo systemctl restart SERVICE_NAME'
```

### Check Deploy Status
```bash
gh run list --repo zenc-cp/claw-stack-jp --limit 5
gh run view RUN_ID --repo zenc-cp/claw-stack-jp --log
```

## 2. Service Management

### All Services Status
```bash
gh workflow run "Run Remote Command" --repo zenc-cp/claw-stack-jp --ref main \
  -f command='systemctl is-active conductor-v2 claw-board zen-console caddy wa-proxy'
```

### Individual Service Control
```bash
# Start / Stop / Restart
sudo systemctl start   <service>
sudo systemctl stop    <service>
sudo systemctl restart <service>

# Service names: conductor-v2, claw-board, zen-console, caddy, wa-proxy
```

### Service Ports
| Service | Port | Health Check |
|---------|------|-------------|
| claw-board | 8090 | `curl -s http://127.0.0.1:8090/` |
| zen-console | 8787 | `curl -s http://127.0.0.1:8787/` |
| caddy | 80/443 | `curl -s https://z3nops.com/board/` |
| wa-proxy | 3000 | `curl -s http://127.0.0.1:3000/` |

## 3. Log Checking

### Recent Logs (last 50 lines)
```bash
sudo journalctl -u conductor-v2 -n 50 --no-pager
sudo journalctl -u claw-board -n 50 --no-pager
sudo journalctl -u zen-console -n 50 --no-pager
sudo journalctl -u caddy -n 50 --no-pager
```

### Follow Logs (live tail)
```bash
sudo journalctl -u conductor-v2 -f
```

### Logs Since Time
```bash
sudo journalctl -u conductor-v2 --since "1 hour ago" --no-pager
sudo journalctl -u conductor-v2 --since "2024-01-15 10:00:00" --no-pager
```

### Error-Only Logs
```bash
sudo journalctl -u conductor-v2 -p err -n 30 --no-pager
```

## 4. Database Query Patterns

### Connect to SQLite
```bash
sqlite3 ~/claw/claw.db
```

### Common Queries
```sql
-- Recent conductor runs
SELECT * FROM conductorlog ORDER BY timestamp DESC LIMIT 20;

-- Agent wrapup summaries
SELECT agent, summary, timestamp FROM agent_wrapup ORDER BY timestamp DESC LIMIT 10;

-- Recent trades
SELECT * FROM trades ORDER BY timestamp DESC LIMIT 10;

-- Brain memory search
SELECT * FROM brain_memory WHERE content LIKE '%keyword%' ORDER BY timestamp DESC LIMIT 10;

-- Skill usage stats
SELECT skill, count(*) as uses FROM skill_usage GROUP BY skill ORDER BY uses DESC;

-- Check DB size
SELECT page_count * page_size as size_bytes FROM pragma_page_count(), pragma_page_size();
```

### Safe Read Pattern (WAL-safe)
```python
import sqlite3
conn = sqlite3.connect('file:~/claw/claw.db?mode=ro', uri=True)
# Read-only, WAL-safe
```

### Write Pattern (with timeout for WAL locking)
```python
import sqlite3
conn = sqlite3.connect('~/claw/claw.db', timeout=10)
# timeout=10s prevents SQLITE_BUSY on WAL contention
```

## 5. Common Fixes

### claw-board 502 (port 8090 not responding)
```bash
# Check if process is running
sudo systemctl status claw-board
# Check the actual error
sudo journalctl -u claw-board -n 30 --no-pager
# Restart
sudo systemctl restart claw-board
# Verify
sleep 2 && curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8090/
```

### zen-console 502 (Hermes WebUI down)
```bash
# Check status and port
sudo systemctl status zen-console
sudo ss -tlnp | grep 8787
# Restart
sudo systemctl restart zen-console
# Verify
sleep 3 && curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8787/
```

### conductor-v2 Stuck (not cycling agents)
```bash
# Check if running
sudo systemctl status conductor-v2
# Check for error loops
sudo journalctl -u conductor-v2 -n 50 --no-pager | tail -20
# Hard restart
sudo systemctl restart conductor-v2
# Verify agent cycles resume
sleep 10 && sudo journalctl -u conductor-v2 -n 5 --no-pager
```

### claw.db Locked (SQLITE_BUSY)
```bash
# Find who's holding the lock
fuser ~/claw/claw.db
# Check WAL file size (large = stuck writer)
ls -lh ~/claw/claw.db-wal
# Force WAL checkpoint (only if no active writers)
sqlite3 ~/claw/claw.db 'PRAGMA wal_checkpoint(TRUNCATE);'
# If still locked, identify and restart the blocking service
```

### Caddy Config Error (site not loading)
```bash
# Validate config
caddy validate --config /etc/caddy/Caddyfile
# Reload (non-destructive)
sudo systemctl reload caddy
# If reload fails, check logs
sudo journalctl -u caddy -n 20 --no-pager
```

## 6. Agent Debugging

### Check Agent Last Run
```bash
# Via conductor log
sqlite3 ~/claw/claw.db "SELECT agent, status, timestamp FROM conductorlog WHERE agent='AGENT_NAME' ORDER BY timestamp DESC LIMIT 5;"
```

### Check Agent Wrapup (summary of what it did)
```bash
sqlite3 ~/claw/claw.db "SELECT agent, summary, timestamp FROM agent_wrapup WHERE agent='AGENT_NAME' ORDER BY timestamp DESC LIMIT 3;"
```

### Agent Scripts Location
```
conductor/agents/trader/exec.py
conductor/agents/hunter/
conductor/agents/scribe/
conductor/agents/sentinel/
conductor/agents/scout/
conductor/agents/ops/
```

### Force Run a Single Agent
```bash
cd ~/claw && python3 conductor/agents/trader/exec.py
cd ~/claw && python3 conductor/agents/sentinel/exec.py
```

### Check Agent State
```bash
# Each agent may have state.json in its directory
cat ~/claw/conductor/agents/trader/state.json 2>/dev/null || echo "No state file"
cat ~/claw/conductor/agents/sentinel/state.json 2>/dev/null || echo "No state file"
```

## 7. Caddy Config Management

### Config Location
```
/etc/caddy/Caddyfile
```

### View Current Config
```bash
cat /etc/caddy/Caddyfile
```

### Edit and Reload
```bash
# Edit (always via repo, never directly on VM)
# After pushing changes:
gh workflow run "Run Remote Command" --repo zenc-cp/claw-stack-jp --ref main \
  -f command='sudo cp ~/claw/caddy/Caddyfile /etc/caddy/Caddyfile && sudo systemctl reload caddy'
```

### Verify Caddy Routes
```bash
curl -s -o /dev/null -w "%{http_code}" https://z3nops.com/board/
curl -s -o /dev/null -w "%{http_code}" https://z3nops.com/console/
```

### Check TLS Certificate
```bash
caddy trust
openssl s_client -connect z3nops.com:443 -servername z3nops.com </dev/null 2>/dev/null | openssl x509 -noout -dates
```

## 8. Quick Health Check (all-in-one)

```bash
echo "=== Services ===" && \
systemctl is-active conductor-v2 claw-board zen-console caddy && \
echo "=== Board ===" && \
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:8090/ && \
echo "=== Console ===" && \
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:8787/ && \
echo "=== External ===" && \
curl -s -o /dev/null -w "HTTP %{http_code}\n" https://z3nops.com/board/ && \
echo "=== DB ===" && \
sqlite3 ~/claw/claw.db "SELECT count(*) || ' conductor logs' FROM conductorlog;" && \
echo "=== Disk ===" && \
df -h / | tail -1
```
