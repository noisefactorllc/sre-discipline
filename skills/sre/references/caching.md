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

A CDN edge or reverse proxy may fail to reach the origin because of network errors, DNS failures, or upstream timeouts. Many configurations then silently serve stale content through directives such as `proxy_cache_use_stale`. The TTL expires and revalidation fails, but the cache serves the stale response without a visible error. Users see old content indefinitely.

**This is the most insidious caching failure mode.** If all edges/proxies are stale simultaneously, suspect a connectivity issue — not just expired caches.

Check edge/proxy error logs for `connect() failed`, `Network unreachable`, `upstream timed out`.

### IPv6 connectivity and cache revalidation

A CDN edge or proxy serves stale content indefinitely when all these conditions apply:

- It resolves an origin AAAA record.
- Its IPv6 connection fails with "Network unreachable".
- Its configuration includes `proxy_cache_use_stale error`.

The Docker network on the edge may lack IPv6 support.

```bash
# Test IPv6 from inside a cache container
docker exec <cache-container> curl -6 -sf https://<origin-domain>/up --connect-timeout 5
```

### Cache key surprises

Cache keys determine which requests share cached content. If the cache key is `$host$uri`, it excludes the query string. Cache-busting parameters such as `?v=2` or `?cb=timestamp` then have no effect. Check the cache configuration before assuming query parameters will bypass it.

### Purge verification from inside vs outside

After purging a cache, don't check by hitting the service directly (bypassing the cache). Check through the cache — the path your users take:

```bash
# Wrong: bypasses CDN, hits origin directly
curl -sf https://<origin-ip>/<path>

# Right: goes through CDN, verifies cache was actually purged
curl -sI https://<domain>/<path> | grep -i x-cache
# Should show MISS on first request after purge
```

### Partial purges on multi-edge CDNs

If you purge only some edges, users at other edges still see stale content. This causes intermittent behavior that depends on location. Always purge ALL edges. Then check ALL edges.

## Cache Invalidation Principles

### Prefer time-based TTLs over event-based purging

Event-based purging ("purge when we deploy") depends on the purge mechanism working. It must reach every cache node before users request stale content. Short TTLs are simpler and let stale content expire automatically.

### Version your static assets

Include a content hash or version in filenames: `app.a3b2c4.js`, not `app.js`. New deploys reference new filenames, so there's nothing to invalidate. Old URLs remain valid (serving old version, which nothing links to). This lets you set long cache TTLs for performance with instant updates.

### Never assume a purge succeeded

Always check by requesting the resource THROUGH the cache after purging.

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
