# System Health — Zenda Stack Monitoring

## Overview
Monitor all Zenda 4.0 services and report their status.

## Health Check
When Zen asks for system status or health:
```bash
echo "=== Zenda 4.0 Health Check $(date) ==="

# Check core services
for svc in claw-core claw-gateway wa-proxy caddy claw-board conductor.timer; do
  if systemctl is-active --quiet "$svc" 2>/dev/null; then
    echo "[OK] $svc"
  else
    echo "[FAIL] $svc"
  fi
done

# Check ports
for port_name in "8080:gateway" "3100:wa-proxy" "8090:zenops" "18000:paywall" "42617:daemon"; do
  port="${port_name%%:*}"
  name="${port_name##*:}"
  if curl -sf --max-time 3 "http://localhost:$port/health" >/dev/null 2>&1 || ss -tlnp | grep -q ":$port "; then
    echo "[OK] $name (port $port)"
  else
    echo "[FAIL] $name (port $port)"
  fi
done

echo ""
echo "Disk: $(df -h / | tail -1)"
echo "Memory: $(free -h | grep Mem)"
echo "Load: $(uptime)"
```

## Trigger Phrases
- "health check" or "status" → Run full health check
- "disk space" → Check disk usage
- "memory" → Check RAM usage
- "what's running" → List active services

## Disk Check
```bash
df -h / /home
du -sh ~/claw/data/* 2>/dev/null | sort -rh | head -10
```
