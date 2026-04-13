# BlacksmithAI v2 — Full Penetration Testing Toolkit

## Overview
BlacksmithAI is ZeroClaw's security offensive toolkit. It orchestrates 20+ CLI tools
through the full penetration testing lifecycle: Recon → Scan → Exploit → Post-Exploit → Report.

All tools run directly on the VM (no Docker). Guard agent's `deep_scan.py` handles the
automated pipeline; this skill gives ZeroClaw interactive access to individual tools.

## Quick Commands

### Full Recon (recommended first step)
```bash
mkdir -p ~/claw/data/reports/security-$(date +%Y%m%d-%H%M%S)
OUTDIR=~/claw/data/reports/security-$(date +%Y%m%d-%H%M%S)

# Subdomain discovery (two sources merged)
subfinder -d TARGET -o $OUTDIR/subs-subfinder.txt -silent
assetfinder --subs-only TARGET >> $OUTDIR/subs-subfinder.txt
sort -u $OUTDIR/subs-subfinder.txt -o $OUTDIR/subdomains.txt

# Live host probing
httpx -l $OUTDIR/subdomains.txt -sc -title -tech-detect -o $OUTDIR/live-hosts.txt -silent

# DNS enumeration
dnsrecon -d TARGET -t std > $OUTDIR/dns-recon.txt 2>&1
```
Report: number of subdomains, live hosts, technologies detected.

### Vulnerability Scan (web target)
```bash
OUTDIR=~/claw/data/reports/security-$(date +%Y%m%d-%H%M%S)
mkdir -p $OUTDIR

# Nuclei — template-based vuln scan
nuclei -u TARGET_URL -severity medium,high,critical -j -o $OUTDIR/nuclei.jsonl -silent

# Nikto — web server vulns
nikto -h TARGET_URL -o $OUTDIR/nikto.txt

# SSL/TLS analysis
sslscan TARGET_HOST > $OUTDIR/sslscan.txt

# Web technology fingerprint
whatweb TARGET_URL > $OUTDIR/whatweb.txt
```

### Port Scan (infrastructure)
```bash
OUTDIR=~/claw/data/reports/security-$(date +%Y%m%d-%H%M%S)
mkdir -p $OUTDIR

# Fast port discovery
masscan TARGET -p1-65535 --rate 1000 -oG $OUTDIR/masscan.txt

# Detailed service scan on discovered ports
nmap -sV -sC -p PORTS TARGET -oN $OUTDIR/nmap.txt

# Service fingerprinting
fingerprintx -t TARGET:PORT > $OUTDIR/fingerprint.txt
```

### Directory Brute-Force
```bash
gobuster dir -u TARGET_URL -w /usr/share/wordlists/dirb/common.txt -o $OUTDIR/dirs.txt -q
```

### WordPress Scan
```bash
wpscan --url TARGET_URL --enumerate vp,vt,u --output $OUTDIR/wpscan.txt
```

### Deep Scan (automated multi-tool pipeline)
```bash
python3 ~/claw/conductor/agents/guard/deep_scan.py --target TARGET --json
```
This runs subfinder→httpx→nikto→sqlmap→gobuster→nmap→nuclei→SSL audit automatically.

### Secret Detection
```bash
python3 ~/claw/conductor/agents/guard/secret_scan.py --path /path/to/code --json
trufflehog git https://github.com/org/repo --json > $OUTDIR/secrets.json
```

## Exploitation (REQUIRES APPROVAL)
These commands MUST NOT be run without explicit "approved" from Zen:

```bash
# Auth brute-force
hydra -L users.txt -P passwords.txt TARGET ssh -t 4

# SQL injection exploitation
sqlmap -u "URL?param=value" --dump --batch

# Windows lateral movement
impacket-psexec DOMAIN/user:password@TARGET
```

## Safety
- Only scan in-scope targets or paid x402 requests
- Never scan private/internal networks without explicit approval
- Exploitation tools require Zen's explicit approval before running
- Always create timestamped report directory
- Rate-limit masscan to 1000 pps on shared infrastructure
