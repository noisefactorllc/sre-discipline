# Caching Reference

Domain-specific traps, diagnostic commands, and checklist items for cache invalidation and CDN operations.

## Pre-flight Items

Add these to your Phase 2 checklist when the operation involves cached content:

- [ ] **All cache layers in the request path identified** — browser, CDN edge, reverse proxy, application cache, DNS resolver
- [ ] **TTLs for each layer documented** — how long will stale content persist without intervention?
- [ ] **Purge mechanism identified** — can you force invalidation? Or must you wait for TTL expiry?
- [ ] **Impact of stale content assessed** — cosmetic (wrong color) or functional (broken JavaScript hitting new API)?
- [ ] **All edge nodes identified** (if CDN) — a partial purge leaves some users seeing stale content depending on which edge they hit

## Traps

### Silent stale serving

When a CDN edge or reverse proxy can't reach the origin (network error, DNS failure, upstream timeout), many configurations serve stale cached content silently via directives like `proxy_cache_use_stale`. The cache appears frozen — the TTL expires, the revalidation attempt fails, and the stale response is served anyway with no visible error. The only symptom is users seeing old content indefinitely.

**This is the most insidious caching failure mode.** If all edges/proxies are stale simultaneously, suspect a connectivity issue — not just expired caches.

Check edge/proxy error logs for `connect() failed`, `Network unreachable`, `upstream timed out`.

### IPv6 connectivity and cache revalidation

If a CDN edge or proxy resolves an origin's AAAA record, attempts an IPv6 connection that fails ("Network unreachable"), and has `proxy_cache_use_stale error` configured, it serves stale content forever. The Docker network on the edge may lack IPv6 support.

```bash
# Test IPv6 from inside a cache container
docker exec <cache-container> curl -6 -sf https://<origin-domain>/up --connect-timeout 5
```

### Cache key surprises

Cache keys determine what's "the same" content. Common surprise: if the cache key is `$host$uri` (without query string), then cache-busting query parameters (`?v=2`, `?cb=timestamp`) have zero effect. Check your cache configuration before assuming query params will bypass the cache.

### Purge verification from inside vs outside

After purging a cache, don't verify by hitting the service directly (bypassing the cache). Verify through the cache — the path your users take:

```bash
# Wrong: bypasses CDN, hits origin directly
curl -sf https://<origin-ip>/<path>

# Right: goes through CDN, verifies cache was actually purged
curl -sI https://<domain>/<path> | grep -i x-cache
# Should show MISS on first request after purge
```

### Partial purges on multi-edge CDNs

If you purge some edges but not all, users routed to unpurged edges still see stale content. The behavior appears intermittent and location-dependent — the worst kind of bug. Always purge ALL edges, and verify ALL edges afterward.

## Cache Invalidation Principles

### Prefer time-based TTLs over event-based purging

Event-based purging ("purge when we deploy") is fragile — it depends on the purge mechanism working, reaching every cache node, and completing before users hit stale content. Short TTLs are simpler and self-healing.

### Version your static assets

Include a content hash or version in filenames: `app.a3b2c4.js`, not `app.js`. New deploys reference new filenames, so there's nothing to invalidate. Old URLs remain valid (serving old version, which nothing links to). This lets you set long cache TTLs for performance with instant updates.

### Never assume a purge succeeded

Always verify by requesting the resource THROUGH the cache after purging.

## Diagnostic Commands

```bash
# Check cache status headers
curl -sI https://<domain>/<path> | grep -i 'x-cache'
# X-Cache-Status: HIT = cached, MISS = fresh from origin, STALE = stale

# Check from a specific CDN edge (bypass DNS routing)
curl -sI --resolve <domain>:443:<edge-ip> https://<domain>/<path> | grep -i 'x-cache'

# Check ETag/Last-Modified to detect staleness
curl -sI https://<domain>/<path> | grep -iE 'etag|last-modified'

# Compare origin vs edge content
origin_etag=$(curl -sI --resolve <domain>:443:<origin-ip> https://<domain>/<path> | grep -i etag)
edge_etag=$(curl -sI --resolve <domain>:443:<edge-ip> https://<domain>/<path> | grep -i etag)
echo "Origin: $origin_etag"
echo "Edge:   $edge_etag"

# Check cache configuration (nginx)
docker exec <cache-container> cat /etc/nginx/conf.d/default.conf | grep -E 'proxy_cache|cache_valid|cache_key|cache_use_stale'
```

## Verification Commands

After cache purge or config change:

```bash
# Verify purge worked (should be MISS on first request)
curl -sI https://<domain>/<path> | grep -i x-cache-status
# Expected: MISS

# Verify on subsequent request (should be HIT with fresh content)
curl -sI https://<domain>/<path> | grep -i x-cache-status
# Expected: HIT

# Verify across all edges
for edge_ip in <ip1> <ip2> <ip3>; do
  status=$(curl -sI --resolve <domain>:443:$edge_ip https://<domain>/<path> | grep -i x-cache-status)
  echo "$edge_ip: $status"
done
```
