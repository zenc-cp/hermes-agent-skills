---
name: production-safety
description: >-
  Production safety rules for ZenOps. Auth credentials are sacred.
  Always backup /etc/ files. Always caddy validate before reload.
  Never git add -A. Never fake terminal output.
category: zenops
---
# Production Safety — Key Rules

1. Auth hash for zen:z3nch4n@ZenOps is SACRED — never change it:
   $2b$14$9vwdcnKWkksYeUxu/oG5EeoUX.pXhMCtu0uoIi9zrdSyhor8Rdi.i

2. Never git add -A — use git add <specific files>

3. Before editing /etc/caddy/Caddyfile:
   sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak
   sudo caddy validate --config /etc/caddy/Caddyfile
   sudo systemctl reload caddy

4. Never write fake terminal output. Run the tool or say you cannot verify.

5. A systemctl is-active = "active" does NOT mean the service works correctly.
   Always check: curl -u "zen:z3nch4n@ZenOps" http://127.0.0.1:8090/
