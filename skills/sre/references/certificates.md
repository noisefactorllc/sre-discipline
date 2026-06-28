# Certificates Reference

Domain-specific traps, diagnostic commands, and checklist items for TLS certificate operations.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves certificates:

- [ ] **Certificate type identified** — ACME auto-managed (Caddy/certbot) or manually provisioned?
- [ ] **All endpoints that serve this domain identified** — a certificate change may need to happen on origin AND every CDN edge
- [ ] **Challenge type compatible with architecture** — HTTP-01 requires the requesting server to receive the challenge. If DNS routing (latency-based, geo) sends challenges to the wrong server, use DNS-01 instead.
- [ ] **No `tls internal` or self-signed certs** — internal CAs are for internal service-to-service communication only. If a user's browser can reach it, it needs a real cert from a public CA.

## Traps

### Self-signed certificates to users

**Never serve self-signed or internal CA certificates to users.** This includes Caddy's `tls internal` directive. Internal CAs break API clients, mobile apps, automated systems, and train users to click through security warnings. If a browser can reach it, it gets a real cert.

### Force-recreating proxy containers wipes cert cache

Caddy and similar ACME-enabled proxies store certificates and ACME account data in volumes or internal state. Running `docker compose up -d --force-recreate` on the proxy container wipes this state. Certificates must be re-issued, and during the re-issuance window (seconds to minutes), users see TLS errors.

Use `docker compose up -d` (without `--force-recreate`) — it only recreates the container if configuration changed.

### ACME challenges and CDN/latency-based routing

HTTP-01 ACME challenges require the certificate authority to reach the requesting server on port 80. If your DNS uses latency-based routing (Route53, Cloudflare, etc.), the challenge request may be routed to a different server than the one requesting the cert. The challenge fails silently, the cert isn't issued, and the old cert eventually expires.

**Fix:** Use DNS-01 challenges for any domain with non-deterministic routing. DNS-01 proves domain ownership via DNS TXT records, not HTTP requests, so routing topology doesn't matter.

```bash
# certbot DNS-01 example (Route53)
certbot certonly --dns-route53 -d example.com
```

### Multi-region certificate gaps

When serving content from multiple edge nodes, every edge must have its own valid certificate for every domain it serves. A CDN edge serving the wrong certificate (or an expired one) causes intermittent errors that depend on which edge the user hits.

**Issue certificates on all edges BEFORE deploying new domain configurations.** Don't deploy the config first and issue certs later — there will be a window where users hit uncertified edges.

## Diagnostic Commands

```bash
# Check what certificate a server is actually serving
echo | openssl s_client -servername <domain> -connect <ip>:443 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# Check certificate from a specific edge (bypass DNS)
echo | openssl s_client -servername <domain> -connect <edge-ip>:443 2>/dev/null | openssl x509 -noout -subject -dates

# Check certificate expiry
echo | openssl s_client -servername <domain> -connect <domain>:443 2>/dev/null | openssl x509 -noout -enddate

# Check Caddy's managed certificates
docker exec <caddy-container> caddy list-certs 2>/dev/null || docker exec <caddy-container> ls /data/caddy/certificates/

# Check certbot certificates
certbot certificates

# Verify full certificate chain
curl -vI https://<domain> 2>&1 | grep -E 'subject:|issuer:|expire'
```

## Verification Commands

After any certificate change, verify from outside — not just from the server:

```bash
# Verify correct domain, valid dates, trusted CA
echo | openssl s_client -servername <domain> -connect <domain>:443 2>/dev/null | openssl x509 -noout -text | grep -E 'Subject:|Issuer:|Not Before:|Not After:'

# Verify from a specific edge
curl -sI --resolve <domain>:443:<edge-ip> https://<domain>/ | head -5

# Verify no certificate warnings (curl will error on bad certs)
curl -sf https://<domain>/up && echo "OK" || echo "CERT OR SERVICE PROBLEM"
```

## Certificate Monitoring

Monitor certificate expiration on every endpoint at least daily. Alert with 14+ days lead time. Monitor the actual certificate *served by the endpoint*, not just what's on disk — a renewed cert that isn't loaded by the server is useless.

```bash
# Quick expiry check (returns days until expiry)
exp=$(echo | openssl s_client -servername <domain> -connect <domain>:443 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
echo $(( ($(date -d "$exp" +%s) - $(date +%s)) / 86400 )) days
```
