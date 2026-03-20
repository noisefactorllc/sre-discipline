# SRE Discipline

A Claude Code plugin that enforces battle-tested SRE operational discipline for infrastructure work.

When activated, it guides Claude through a rigorous checklist procedure for deployments, rollbacks, migrations, cache invalidation, certificate management, DNS changes, firewall rules, container orchestration, and incident response.

Every rule in this procedure exists because someone learned it the hard way.

## What it does

- **Classifies operations** as planned work or active incidents, with different procedures for each
- **Enforces a 5-phase checklist**: Research, Pre-flight, Execution, Short-term follow-through, Long-term follow-through
- **Loads domain-specific references** for containers, certificates, DNS, caching, and firewalls
- **Blocks common anti-patterns**: fix cascading, commit thrashing, scope creep, premature rollback
- **Requires verification** before and after every change

## Install

```
/plugins install sre-discipline
```

Or add to your project's `.claude/plugins.json`:

```json
{
  "plugins": [
    {
      "name": "sre-discipline",
      "source": "noisedeck/sre-discipline"
    }
  ]
}
```

## When it activates

The skill triggers whenever Claude is performing any operation that touches:

- Docker containers or compose files
- Reverse proxies (Caddy, nginx, Traefik)
- SSL/TLS certificates or ACME
- CDN caches or static asset pipelines
- DNS records
- Firewall rules or network security
- Deployment pipelines
- Production incidents

## Reference docs

The plugin includes domain-specific reference documents that are loaded contextually:

| Reference | Covers |
|-----------|--------|
| `containers.md` | Docker traps, env var loss, network integrity, stale containers |
| `certificates.md` | ACME challenges, force-recreation cert wipe, multi-region gaps |
| `dns.md` | TTL blindness, missing AAAA records, premature record deletion |
| `caching.md` | Silent stale serving, cache key surprises, partial CDN purges |
| `firewalls.md` | ufw vs Docker iptables, DOCKER-USER chain, rule persistence |
| `incidents.md` | Incident response procedure, anti-patterns that escalate outages |

## License

MIT
