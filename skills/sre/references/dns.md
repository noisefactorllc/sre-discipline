# DNS Reference

Domain-specific traps, diagnostic commands, and checklist items for DNS operations.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves DNS:

- [ ] **Current TTL noted** — how long will stale records persist after the change?
- [ ] **Both A and AAAA records planned** — IPv6-only users (common on mobile) can't reach IPv4-only services
- [ ] **Old records preserved during migration** — Keep old records until the new destination passes verification and at least 2x TTL elapses.
- [ ] **TTL lowered in advance** (if migrating) — lower TTL to 300s at least 2x the current TTL before the migration window

## Traps

### Missing AAAA records

**Always create both A (IPv4) and AAAA (IPv6) records.** Users on IPv6-only networks, increasingly common on mobile carriers, cannot reach IPv4-only services. A service can work on your IPv4 connection while failing for those users. Monitoring will not detect this unless it checks from IPv6.

### TTL blindness

DNS changes are not instant. If your TTL is 300 seconds, changes propagate within ~5 minutes. If it's 86400 seconds (24 hours), users may hit the old address for up to a full day. Your local resolver cache may show the new record while most of the internet still sees the old one.

Before a migration:
1. Check the current TTL
2. Lower it to 300s well in advance (at least 2x the current TTL before the change window)
3. Wait for the old TTL to expire
4. Make the change
5. After verification, you can raise the TTL back if desired

### Premature record deletion

Don't delete old DNS records during a migration until you're certain nothing still needs them:
1. Add the new records first
2. Check the new destination works
3. Wait at least 2x the old TTL
4. Then remove the old records

Deleting old records immediately means any cached resolver still pointing to the old IP will fail — and you can't speed up the cache expiry.

### Resolver cache confusion

Your local resolver may cache different results than what users see. When checking DNS changes, query external resolvers directly:

```bash
# Query specific public resolvers
dig @8.8.8.8 <domain> A
dig @1.1.1.1 <domain> AAAA
dig @9.9.9.9 <domain> A

# Check TTL remaining on cached record
dig <domain> +noall +answer
```

## Diagnostic Commands

```bash
# Current A records
dig <domain> A +short

# Current AAAA records
dig <domain> AAAA +short

# Full record details including TTL
dig <domain> +noall +answer

# Check from multiple resolvers
for ns in 8.8.8.8 1.1.1.1 9.9.9.9; do echo "=== $ns ==="; dig @$ns <domain> A +short; done

# Check nameservers
dig <domain> NS +short

# Check all record types
dig <domain> ANY +noall +answer

# Trace resolution path
dig <domain> +trace
```

## Verification Commands

After any DNS change:

```bash
# Verify A record resolves to expected IP
dig <domain> A +short
# Should return: <expected-ip>

# Verify AAAA record exists
dig <domain> AAAA +short
# Should return: <expected-ipv6>

# Verify from external resolvers (not local cache)
dig @8.8.8.8 <domain> A +short
dig @1.1.1.1 <domain> A +short

# Verify the service actually responds at the new IP
curl -sf --resolve <domain>:443:<new-ip> https://<domain>/up

# Check propagation across resolvers
for ns in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222; do
  echo "$ns: $(dig @$ns <domain> A +short)"
done
```
