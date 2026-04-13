# Security Recon — DNS & Network Tools

## DNS Lookup
```bash
dig TARGET_DOMAIN ANY +short
dig TARGET_DOMAIN MX +short
dig TARGET_DOMAIN TXT +short
dig TARGET_DOMAIN NS +short
```

## HTTP Check
```bash
curl -sI -o /dev/null -w "HTTP %{http_code} | %{time_total}s | %{redirect_url}" TARGET_URL
```

## Whois
```bash
whois TARGET_DOMAIN 2>/dev/null | grep -E "Registrar|Creation|Expiry|Name Server"
```

## Trigger Phrases
- "dig DOMAIN" or "dns DOMAIN" → DNS lookup
- "check URL" or "http check URL" → HTTP response check
