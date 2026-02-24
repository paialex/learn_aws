# AWS ALB Visual Guide: Architecture Patterns (with API Gateway)

This guide is **ALB-centric** and also covers **API Gateway API types (REST/HTTP/WebSocket)** because those choices are part of modern edge architecture and were explicitly requested.

In practice, strong AWS API platforms often use:
- **API Gateway** for API product management, auth policies, and API-specific controls.
- **ALB** for advanced Layer-7 routing to private services in VPCs (ECS/EKS/EC2/Lambda targets).

## 1. Core Functionalities

![Core Functionalities](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/core-functionalities.png)

### 1.1 Request Routing
- ALB provides content-based routing using listener conditions: `host-header`, `path-pattern`, `http-header`, `http-request-method`, `query-string`, and `source-ip`.
- ALB can route to multiple target groups (`instance`, `ip`, `lambda`) and supports microservice fan-out.
- API Gateway can route by resource/route and then forward to public integrations or private integrations (via VPC links).

### 1.2 Authentication and Authorization
- ALB can offload user authentication using **OIDC** and **Amazon Cognito** (`authenticate-oidc` / `authenticate-cognito`).
- API Gateway supports IAM, Cognito, JWT authorizers (HTTP APIs), and Lambda authorizers.
- Recommended split:
  - API Gateway for API authorization policy and consumer segmentation.
  - ALB for app-session oriented auth where needed.

### 1.3 Throttling
- ALB does **not** provide API product-level per-client usage plans.
- For L7 request-rate protection in ALB architectures, use **AWS WAF rate-based rules**.
- For API product throttling and quotas, REST APIs in API Gateway support API keys, usage plans, and per-client throttling.

### 1.4 Protocol Translation
- ALB supports HTTP/1.1 and can use **HTTP/2 or gRPC** toward targets (HTTPS listener required for HTTP/2/gRPC target protocol versions).
- ALB also supports native WebSocket pass-through on HTTP/HTTPS listeners.
- API Gateway handles API protocol models (REST/HTTP/WebSocket) at the edge and can forward to ALB-backed private services.

### 1.5 Payload Transformation
- ALB supports listener rule transforms:
  - `host-header-rewrite`
  - `url-rewrite` (path/query rewrite)
- API Gateway REST APIs support request/response mapping and request body transformation.
- API Gateway HTTP APIs support parameter mapping (lighter transformation model).

## 2. Use Cases: REST APIs vs HTTP APIs vs WebSocket APIs

![API Type Use Cases](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/api-type-use-cases.png)

| API Type | Choose It When | Avoid It When | ALB Fit |
| --- | --- | --- | --- |
| REST API | You need API keys, usage plans, per-client throttling, request validation, richer governance | You want minimum latency/cost with simpler features | Strong fit for private integrations to ALB for internal services |
| HTTP API | You need low-latency, lower-cost RESTful APIs with JWT/OIDC/Cognito basics | You need advanced REST API governance features | Strong fit for private integration with ALB via VPC link |
| WebSocket API | You need bidirectional, real-time communication (chat, live dashboards, alerts) | Request/response-only APIs | ALB can serve websocket-capable backends, but API Gateway WebSocket manages connection lifecycle at scale |

### Practical selection rule
- Start with **HTTP API** for most new stateless APIs.
- Use **REST API** when governance/monetization/security controls require it.
- Use **WebSocket API** only for real-time bidirectional use cases.
- Put **ALB behind API Gateway** when you need private, advanced L7 routing for many internal services.

## 3. Architectural Patterns

### 3.1 Centralized Edge Gateway Pattern

![Centralized Edge Gateway](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/centralized-edge-gateway.png)

**Intent:** One edge entry point for authentication, policy enforcement, and consistent routing.

**Traffic flow:**
1. Clients send HTTPS traffic to WAF + API Gateway.
2. Gateway applies Cognito/JWT/Lambda authorizer checks.
3. Public and private routes are split to Lambda and ALB-backed VPC services.

**Best for:** Enterprise platforms with many API consumers and strong governance.

### 3.2 Backend-for-Frontend (BFF) Pattern

![BFF Pattern](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/bff-pattern.png)

**Intent:** Separate API contracts by client channel (web/mobile/partner).

**Traffic flow:**
1. Clients hit one gateway domain.
2. Route-level segmentation forwards requests to dedicated BFF services.
3. BFFs call shared private services via ALB and/or dedicated data stores.

**Best for:** Different UX channels with different response shapes, caching needs, and release cadence.

### 3.3 Microgateway Pattern

![Microgateway Pattern](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/microgateway-pattern.png)

**Intent:** Domain-owned gateways to reduce coupling and improve team autonomy.

**Traffic flow:**
1. Entry gateway handles coarse routing.
2. Domain gateways (orders/billing/search) enforce domain-specific auth and controls.
3. Each domain connects to its own ALB/service plane.

**Best for:** Large organizations with many teams and bounded contexts.

### 3.4 Strangler Fig Pattern

![Strangler Fig Pattern](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/strangler-fig-pattern.png)

**Intent:** Incrementally replace a monolith without a big-bang rewrite.

**Traffic flow:**
1. API Gateway is the stable contract for clients.
2. Legacy routes forward to monolith behind ALB.
3. New routes are carved out to modern services and Lambda functions.
4. Route ownership gradually shifts from legacy to modern services.

**Best for:** Controlled modernization with low migration risk.

## 4. Architectural Anti-Patterns

### 4.1 Anti-Pattern: Monolithic Gateway (Single Point of Failure / Blast Radius)

![Monolithic Gateway Anti-Pattern](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/anti-monolithic-gateway.png)

**Why this fails:**
- One gateway deployment/team owns every domain.
- One shared backend plane couples unrelated services.
- Any policy/config/regression event can impact all APIs at once.

**Fix:** Split by bounded context (domain gateways/BFF), isolate failure domains, and decouple deployment pipelines.

### 4.2 Anti-Pattern: Deep Business Logic Embedded in the Gateway

![Business Logic in Gateway Anti-Pattern](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/anti-business-logic-gateway.png)

**Why this fails:**
- Gateway becomes orchestration engine + policy engine + transformation hub.
- Tight coupling increases latency and release risk.
- Harder testing, weaker ownership boundaries.

**Fix:** Keep gateway "thin" (auth, routing, light transformation). Move domain logic to backend services.

### 4.3 Anti-Pattern: Missing Timeout/Retry Configuration

![No Timeout Retry Anti-Pattern](/Users/alexandrupascan/.codex/worktrees/c2ad/learn_aws/alb-visual-guide/generated-diagrams/anti-no-timeout-retry.png)

**Why this fails:**
- Slow downstreams create request pileups.
- Clients retry blindly, causing retry storms.
- End-to-end latency and error rates spike under load.

**Fix:**
- Set explicit timeouts at every hop.
- Use bounded retries with jitter/backoff.
- Add circuit-breaker and load-shedding behavior.
- Ensure idempotency for retryable operations.

## References

- [What is an Application Load Balancer?](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [Condition types for ALB listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-condition-types.html)
- [Transforms for ALB listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-transforms.html)
- [Authenticate users using an Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html)
- [Listeners for your Application Load Balancer (WebSockets, HTTP/2)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)
- [Target groups for ALB (HTTP/2, gRPC protocol versions)](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)
- [Use Lambda functions as ALB targets](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/lambda-functions.html)
- [Applying rate limiting in AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based-request-limiting.html)
- [Choose between REST APIs and HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)
- [API Gateway use cases](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-overview-developer-experience.html)
- [Create private integrations for HTTP APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-private.html)
- [Amazon API Gateway REST APIs now supports private integration with ALB (Nov 21, 2025)](https://aws.amazon.com/about-aws/whats-new/2025/11/api-gateway-rest-apis-integration-load-balancer/)
