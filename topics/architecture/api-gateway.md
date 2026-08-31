# API Gateway

## Core idea

An API Gateway is the controlled edge entry point for application traffic. It hides backend topology from clients and centralizes cross-cutting traffic, security, resilience, deployment, and observability concerns. Business logic should remain in downstream services.

## Mental model

```text
Internet
  ↓
CDN / WAF / DDoS
  ↓
Load Balancer
  ↓
API Gateway
  ├─ Routing / discovery / load balancing
  ├─ TLS / authentication / authorization
  ├─ Tenant and data-scope enforcement
  ├─ Rate limits / quotas / concurrency limits
  ├─ Timeout / retry / circuit breaker
  ├─ Canary / blue-green traffic control
  ├─ Request/response transformation
  ├─ CORS / security headers
  ├─ Logging / metrics / tracing
  └─ Health checks / connection draining
  ↓
Backend Services
```

## Why it exists

The gateway provides a stable external API boundary while backend services can change location, instance count, protocol, version, and deployment strategy without forcing clients to understand internal topology.

## Important functionalities

### Traffic management

- Path, host, method, header, query, and version-based routing.
- Route priorities and fallback routes.
- Load balancing: round robin, least connections, weighted, random, hashing, zone/region-aware routing.
- Service discovery and dynamic backend registration.
- Health-aware routing and failover.

### Security

- TLS termination and certificate management.
- OAuth 2.0 / OpenID Connect, JWT, API keys, mTLS, and cookie authentication as appropriate.
- Token/signature/issuer/audience/expiry validation.
- Role, scope, claim, feature, tenant, and API authorization.
- IP/CIDR filtering.
- CORS and security headers.
- WAF integration and DDoS protection at the edge.
- Request-size and header-size limits.
- Audit logging for security-sensitive operations.

### Multi-tenancy and data scope

- Establish caller identity and tenant context.
- Validate coarse resource/data-scope access before forwarding.
- Prevent obvious cross-tenant or cross-advertiser access.
- Propagate trusted identity/context downstream.
- Keep domain-specific authorization in the owning service.

### Resilience and protection

- Per-route timeouts and deadline propagation.
- Retries for safe transient failures only; avoid unsafe retries of non-idempotent operations unless idempotency is guaranteed.
- Circuit breakers.
- Bulkheads and per-route concurrency limits.
- Rate limiting and quotas.
- Traffic throttling under backend pressure.
- Graceful shutdown and connection draining.
- Maintenance mode where operationally required.

### Deployment and traffic control

- Canary deployments.
- Blue/green deployments.
- Percentage-based traffic splitting.
- Header, user, tenant, cookie, IP, region, or cohort-based routing.
- Rapid rollback by changing traffic distribution.
- API version routing and deprecation handling.

### Request and response handling

- Header injection/removal.
- Path and query transformation.
- Host and protocol transformation.
- Response-header normalization and security headers.
- Error-envelope normalization at the transport level.
- Protocol translation such as REST/JSON to gRPC where justified.
- Request aggregation for simple, well-bounded use cases.

### Performance

- Response caching for suitable stable/read-only data.
- Compression such as gzip/Brotli.
- Connection pooling and upstream connection management.
- Streaming support for large files and streaming APIs.
- WebSocket and SSE support where required.
- Optional request deduplication for carefully selected idempotent operations.

### API governance

- API versioning.
- Deprecation and retirement policies.
- Basic schema/contract enforcement.
- Content-type and required-header validation.
- Client/consumer identification.
- Per-client quotas and policies.

### Observability

- Structured request logging.
- Correlation IDs.
- OpenTelemetry distributed tracing.
- Request/route/upstream latency metrics, especially P50/P95/P99.
- Request, 4xx, 5xx, timeout, retry, circuit-breaker, and rate-limit metrics.
- Active connection and resource-utilization metrics.
- Trace/context propagation to downstream services.

## Production implications

The gateway should remain relatively thin. It is appropriate for cross-cutting policies that apply consistently across services, but it should not become a second monolith.

Do not move domain rules such as "can this user cancel this specific order?" into the gateway. The owning service must remain authoritative for business authorization and invariants.

Do not blindly enable retries. A retry can duplicate side effects when the first request succeeded but its response was lost. Prefer idempotent operations or explicit idempotency keys for retryable writes.

Do not buffer large payloads unnecessarily. Streaming and backpressure matter for large uploads/downloads and realtime APIs.

Aggregation, protocol translation, caching, and advanced traffic policies should be introduced only when there is a concrete requirement because each increases gateway coupling and operational complexity.

For high availability, run multiple gateway instances behind an external load balancer. Gateway instances should be stateless except for explicitly shared infrastructure such as distributed rate-limit or authorization caches.

## Common misconceptions

- **The gateway should contain all authorization.** Wrong. It can enforce coarse access policies, but domain authorization belongs to the service that owns the resource.
- **Every API call should be retried.** Wrong. Retries are safe only when failure semantics and idempotency are understood.
- **The gateway should implement business orchestration.** Wrong. Heavy workflows belong in application services/workflow engines.
- **A gateway is just a reverse proxy.** Incomplete. Routing is the core, but production gateways commonly provide security, resilience, traffic control, and observability.
- **More gateway features are always better.** Wrong. Gateway complexity directly increases blast radius because every request passes through it.

## Related concepts

- Reverse proxy
- Load balancer
- Backend-for-Frontend (BFF)
- Service discovery
- OAuth 2.0 / OpenID Connect
- Rate limiting
- Circuit breaker
- Bulkhead isolation
- Canary deployment
- Blue/green deployment
- OpenTelemetry
- WAF

## Open questions

- Which gateway capabilities should be mandatory platform defaults versus opt-in route policies?
- Where should the boundary between API Gateway and BFF/aggregation be established?
- Which policies require distributed state such as Redis?
