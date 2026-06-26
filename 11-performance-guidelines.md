# 11 — Performance Guidelines

Latency targets, caching, compression, pagination, and monitoring.

---

## Latency Targets

| Endpoint type | Target (p95) | Maximum |
|---------------|-------------|---------|
| Simple GET (single resource) | < 100ms | 300ms |
| List / paginated collection | < 200ms | 500ms |
| Complex query (filtered, sorted, joined) | < 300ms | 800ms |
| POST / write operations | < 300ms | 500ms |
| Report generation / heavy aggregation | < 2000ms | 5000ms |
| File upload | < 5000ms | 30000ms |

**Note:** Targets exclude cold starts and first-connection overhead. Measure from the service boundary, not the client.

If an endpoint cannot meet these targets, investigate before shipping — not after.

---

## Pagination

All collection endpoints must be paginated. Never return unbounded collections.

```
GET /api/v1/users?page=1&per_page=20
```

**Rules:**
- Default `per_page`: 20
- Maximum `per_page`: 100
- Exceeding the maximum returns `422`
- All list endpoints paginate — no exceptions

**Why:** An unbounded `SELECT *` on a growing table will eventually bring down the service.

---

## Database Query Performance

### Avoid N+1 Queries

Load relationships eagerly, not inside loops.

```php
// Bad — N+1
$users = User::all();
foreach ($users as $user) {
    echo $user->role->name; // separate query per user
}

// Good — eager loading
$users = User::with('role')->paginate(20);
```

### Index Strategy

- Index every column used in `WHERE`, `ORDER BY`, or `JOIN`.
- Add composite indexes for multi-column filter + sort combinations.
- Never filter or sort on unindexed columns in production-facing endpoints.

### Query Limits

- Always add `LIMIT` — even for admin endpoints.
- Never expose `SELECT *` — select only the columns needed.
- Use `EXISTS` instead of `COUNT` when checking for presence.

---

## Caching

### When to Cache

| Data type | Cache duration | Strategy |
|-----------|---------------|---------|
| Reference/lookup data (roles, statuses, countries) | 60 seconds | In-memory / Redis |
| User profile | 30 seconds | Redis |
| Paginated list | 10–30 seconds | Redis (keyed by query params) |
| Config / feature flags | 5 minutes | Redis |
| Static/rarely-changing data | 1 hour | Redis |

### Cache Key Pattern

```
{service}:{version}:{resource}:{id or hash}
auth:v1:user:1
catalog:v1:categories:all
catalog:v1:products:page=1:per_page=20:status=active
```

### Cache Invalidation

Invalidate on write:

```
PATCH /users/1 → invalidate cache key auth:v1:user:1
POST  /users   → invalidate auth:v1:users:*
```

### HTTP Caching Headers

For cacheable GET responses:

```http
Cache-Control: public, max-age=30
ETag: "abc123"
Last-Modified: Thu, 26 Jun 2026 10:30:00 GMT
```

For private or dynamic responses:

```http
Cache-Control: no-store
```

---

## Compression

- Enable Gzip or Brotli compression on all API responses.
- Minimum response size for compression: 1KB.
- Set `Content-Encoding: gzip` when compressing.

```http
Accept-Encoding: gzip, deflate, br
Content-Encoding: gzip
```

---

## Async Processing for Heavy Operations

Endpoints that exceed 500ms should be made asynchronous:

```json
// POST /api/v1/reports — response (202 Accepted)
{
    "success": true,
    "message": "Report generation started.",
    "data": {
        "job_id": "job_abc123",
        "status": "queued",
        "status_url": "/api/v1/jobs/job_abc123"
    }
}
```

Client polls the status endpoint:

```
GET /api/v1/jobs/job_abc123
→ { "status": "processing" }
→ { "status": "completed", "download_url": "..." }
```

**Use for:** report exports, bulk imports, email sends, PDF generation.

---

## Rate Limiting

| Tier | Limit |
|------|-------|
| Anonymous | 60 req/min |
| Authenticated user | 1000 req/min |
| Service token | 5000 req/min |
| Heavy endpoints (reports, exports) | 10 req/min |

Rate limit headers on every response:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1719393600
```

---

## Connection Pooling

- Configure database connection pools to match expected concurrency.
- Use persistent connections — avoid reconnecting per request.
- Monitor active connections — alert when approaching pool limit.

| Service load | Pool size guideline |
|-------------|---------------------|
| Low (< 50 req/s) | 10–20 connections |
| Medium (50–200 req/s) | 20–50 connections |
| High (> 200 req/s) | 50–100 connections |

---

## Monitoring

Track these metrics per service in production:

| Metric | Alert threshold |
|--------|----------------|
| p95 latency | > target for endpoint type |
| p99 latency | > 2× target |
| Error rate (5xx) | > 1% |
| Error rate (4xx) | > 5% |
| Request queue depth | > 100 pending |
| DB query time | > 200ms average |
| Cache hit rate | < 80% (for cached endpoints) |

### Recommended Tools

| Tool | Purpose |
|------|---------|
| Prometheus + Grafana | Metrics and dashboards |
| OpenTelemetry | Distributed tracing |
| Jaeger / Zipkin | Trace visualization |
| Sentry | Error tracking and alerting |
| Datadog / New Relic | Managed full-stack observability |

---

## Performance Review Checklist

Before shipping a new endpoint:

- [ ] p95 latency tested under expected load
- [ ] No N+1 queries (check with query log)
- [ ] Pagination enforced
- [ ] Indexes confirmed for all filter/sort columns
- [ ] Cache applied for read-heavy endpoints
- [ ] Compression enabled
- [ ] Rate limiting configured
- [ ] Heavy operations moved to async queue
- [ ] Monitoring dashboard updated
