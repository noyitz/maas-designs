# Design Proposal: vLLM Semantic Router (vSR) Integration with Models-as-a-Service (MaaS)

## Phase 1: Simplified Integration

**Version**: 1.2
**Document Status**: Draft
**Date**: February 2026
**Author**: Noy Itzikowitz
**Previous Versions**: [v1.0](https://github.com/noyitz/maas-designs) (Dec 2025), [v1.1](https://github.com/noyitz/maas-designs) (Jan 2026)

---

## Executive Summary

This document defines the **Phase 1** integration strategy for vLLM Semantic Router (vSR) with the Models-as-a-Service (MaaS) platform. Phase 1 prioritizes a minimal, production-ready integration that gets vSR running behind MaaS security and QoS policies with the least amount of new development.

**Key design decisions for Phase 1:**

- **AuthPolicy protects vSR access** -- a single authorization check determines whether the user can use the semantic router. No per-model RBAC at the gateway level.
- **RateLimitPolicy provides QoS protection** -- rate limiting applies to the entire vSR endpoint to prevent DDoS and abuse. It is not used for per-model accounting or billing.
- **No model lists or quotas in headers** -- vSR receives no `X-Accessible-Models` or `X-Model-Quotas` headers. It routes freely based on its own semantic classification and configured model pool.
- **Internal and external models** -- vSR can route to both internal KServe models and external providers (AWS Bedrock, Azure OpenAI, Google Vertex AI, etc.) through a unified egress architecture. The routing decision is transparent to the client.
- **Observability captures model selection and token usage** -- the actual model selected by vSR and the tokens consumed are emitted as Prometheus metrics for monitoring and future billing readiness. This is capture-only, not enforcement. External model usage is captured with the same metrics, including provider labels.

**What Phase 1 defers to later phases:**
- Per-model RBAC and quota-constrained routing (Phase 2)
- Token-based rate limiting via Limitador (Phase 2)
- Smart cache invalidation and usage tracking APIs (Phase 2)
- Dynamic billing and cost allocation (Phase 3)
- Per-provider cost tracking and budget enforcement (Phase 3)

---

## 1. Architecture Overview

### 1.1 Phase 1 Design Principles

- **Simplicity over completeness**: Ship a working integration with minimal new components
- **Security first**: Users must authenticate before reaching vSR
- **QoS over accounting**: Rate limits prevent abuse, not track spend
- **Observability from day one**: Capture routing decisions and token usage in Prometheus so future phases have data to build on
- **No header pollution**: Keep the request flow clean -- vSR operates with its own intelligence, unconstrained by gateway-injected model lists

### 1.2 Phase 1 vs. Previous Proposals

| Aspect | v1.0 (Dec 2025) | v1.1 (Jan 2026) | v1.2 Phase 1 (Feb 2026) |
|--------|-----------------|-----------------|------------------------|
| Auth scope | Per-model (Phase 3 re-auth) | Per-model (via X-Accessible-Models) | **vSR access only** (binary: allowed/denied) |
| Rate limiting | Per-model + token quotas | Per-model + token quotas via MaaS API | **Entire vSR endpoint** (QoS/DDoS only) |
| Model constraints | Header injection + fallback | X-Accessible-Models + X-Model-Quotas | **None** -- vSR routes freely |
| Fallback logic | Phase 4.5 circuit breaker | Implicit via constrained selection | **Not applicable** -- no model constraints to fall back from |
| Usage tracking | Phase 5 smart cache + billing API | Usage tracking API + cache invalidation | **Prometheus metrics only** (capture, not enforce) |
| RHCL changes | WASM/Lua circuit breaker | No changes | **No changes** |
| MaaS API changes | Multiple new endpoints | accessible-models + usage APIs | **No new endpoints** |
| vSR changes | ExtProc + fallback API | Header parsing + constrained selection | **Metric emission only** |
| External models | Not addressed | Not addressed | **Egress support for AWS Bedrock, Azure OpenAI, etc.** |

### 1.3 Current Platform Architectures

#### MaaS Platform

```mermaid
graph TB
    subgraph "Client Layer"
        Client[Client Applications<br/>with Service Account Token]
    end

    subgraph "Gateway Layer"
        GatewayAPI[maas-default-gateway<br/>All Traffic Entry Point]
        Envoy[Envoy Proxy]
    end

    subgraph "RHCL Policy Engine"
        Kuadrant[Kuadrant<br/>Policy Attachment]
        Authorino[Authorino<br/>Authentication Service]
        Limitador[Limitador<br/>Rate Limiting Service]
    end

    subgraph "Model Serving"
        RHOAI[RHOAI Platform]
        Models[LLM Models<br/>Qwen, Granite, Llama]
    end

    Client --> GatewayAPI
    GatewayAPI --> Envoy
    Envoy --> Kuadrant
    Kuadrant --> Authorino
    Kuadrant --> Limitador
    Envoy --> RHOAI
    RHOAI --> Models
```

#### vSR Platform

```mermaid
graph TB
    subgraph "Proxy Layer"
        Envoy[Envoy Proxy<br/>:8888]
    end

    subgraph "Processing Layer"
        ExtProc[Semantic Router<br/>ExtProc Server<br/>:50051]

        subgraph "Router Components"
            Classifier[BERT Classifier<br/>ModernBERT]
            PIIDetector[PII Detector]
            JailbreakGuard[Jailbreak Guard]
            Cache[Semantic Cache]
            ToolsSelector[Tools Selector]
        end
    end

    subgraph "Internal Models (KServe)"
        Model1[Math Specialist]
        Model2[Creative Model]
        Model3[Code Generator]
    end

    subgraph "External Models (Egress)"
        Bedrock[AWS Bedrock<br/>Claude, Titan]
        AzureOAI[Azure OpenAI<br/>GPT-4]
        ExtAPI[Other Providers]
    end

    Envoy <--> ExtProc
    ExtProc --> Classifier
    ExtProc --> PIIDetector
    ExtProc --> JailbreakGuard
    ExtProc --> Cache
    ExtProc --> ToolsSelector

    Envoy --> Model1
    Envoy --> Model2
    Envoy --> Model3
    Envoy --> Bedrock
    Envoy --> AzureOAI
    Envoy --> ExtAPI
```

---

## 2. Phase 1 Integration Architecture

### 2.1 Integration Flow Overview

Phase 1 implements a **three-phase flow**: authenticate, protect, route, execute, and observe.

```mermaid
graph TB
    subgraph "Phase 1: Authentication & QoS ✅ EXISTING"
        P1A[AuthPolicy: Can user access vSR?<br/>Authorino validates SA token]
        P1B[RateLimitPolicy: QoS Protection<br/>Limitador enforces request rate limits]
    end

    subgraph "Phase 2: Semantic Routing ✅ EXISTING vSR"
        P2[vSR ExtProc<br/>Classification + Security + Model Selection<br/>No external constraints]
    end

    subgraph "Phase 3: Execution & Observability"
        P3A{Internal or<br/>External Model?}
        P3B[KServe Execution ✅ EXISTING]
        P3C[External Provider Egress 🆕 NEW<br/>AWS Bedrock / Azure OpenAI / etc.]
        P3D[Prometheus Metric Capture 🆕 NEW<br/>Model + provider + tokens consumed]
    end

    P1A --> P1B
    P1B --> P2
    P2 --> P3A
    P3A -->|Internal| P3B
    P3A -->|External| P3C
    P3B --> P3D
    P3C --> P3D
```

**Legend:**
- ✅ **EXISTING**: Components that exist today and require no changes (or minimal config)
- 🆕 **NEW**: New components to be implemented

### 2.2 Detailed Sequence

```mermaid
sequenceDiagram
    participant Client
    participant Gateway as maas-default-gateway
    participant Authorino
    participant Limitador
    participant vSR as vSR ExtProc
    participant KServe as Selected Model
    participant Prometheus

    Note over Client,Prometheus: Phase 1: Authentication & QoS

    Client->>Gateway: POST /v1/chat/completions<br/>Authorization: Bearer sa-token-xyz

    Gateway->>Authorino: AuthPolicy: Validate SA token<br/>+ resolve user identity
    Authorino->>Authorino: Kubernetes TokenReview<br/>+ MaaS API tier lookup
    Authorino-->>Gateway: Auth Success<br/>X-User-ID: user-123<br/>X-Tier: premium

    Gateway->>Limitador: RateLimitPolicy: Check request rate<br/>Counter: user-123 (or tier-based)
    Limitador-->>Gateway: OK (within limits)

    Note over Client,Prometheus: Phase 2: Semantic Routing (vSR operates freely)

    Gateway->>vSR: ExtProc: process request<br/>Headers: X-User-ID, X-Tier<br/>Body: user prompt

    vSR->>vSR: 1. PII Detection & Redaction
    vSR->>vSR: 2. Jailbreak Detection

    alt Jailbreak Detected
        vSR-->>Gateway: 403 Forbidden
        Gateway-->>Client: 403 Security Violation
    else Request is Safe
        vSR->>vSR: 3. Semantic Classification (ModernBERT)
        vSR->>vSR: 4. Semantic Cache Check
        alt Cache Hit
            vSR-->>Gateway: Cached response
            Gateway-->>Client: Response
        else Cache Miss
            vSR->>vSR: 5. Model Selection<br/>(unconstrained, based on classification)
            vSR-->>Gateway: Route to selected model<br/>Set Host header + inject:<br/>X-VSR-Model-Selected: llama3-70b<br/>X-VSR-Category: mathematics
        end
    end

    Note over Client,Prometheus: Phase 3: Execution & Observability (Internal Model)

    Gateway->>KServe: Forward request to llama3-70b
    KServe-->>Gateway: Model response<br/>usage.total_tokens: 1500

    Gateway-->>Client: Response

    Note over Prometheus: Async metric emission
    Gateway->>Prometheus: vsr_requests_total{model="llama3-70b", provider="kserve", tier="premium", category="mathematics"}
    Gateway->>Prometheus: vsr_tokens_consumed_total{model="llama3-70b", provider="kserve", tier="premium"} += 1500
```

#### External Model Sequence (AWS Bedrock Example)

```mermaid
sequenceDiagram
    participant Client
    participant Gateway as maas-default-gateway
    participant Authorino
    participant Limitador
    participant vSR as vSR ExtProc
    participant Bedrock as AWS Bedrock<br/>(External)
    participant Prometheus

    Note over Client,Prometheus: Phase 1: Authentication & QoS (identical to internal flow)

    Client->>Gateway: POST /v1/chat/completions<br/>Authorization: Bearer sa-token-xyz

    Gateway->>Authorino: AuthPolicy: Validate SA token
    Authorino-->>Gateway: Auth Success<br/>X-User-ID: user-123, X-Tier: premium

    Gateway->>Limitador: RateLimitPolicy: Check request rate
    Limitador-->>Gateway: OK (within limits)

    Note over Client,Prometheus: Phase 2: Semantic Routing (vSR selects external model)

    Gateway->>vSR: ExtProc: process request

    vSR->>vSR: 1. PII Detection & Redaction
    vSR->>vSR: 2. Jailbreak Detection
    vSR->>vSR: 3. Semantic Classification: advanced_reasoning
    vSR->>vSR: 4. Model Selection: bedrock/claude-sonnet<br/>(best fit for advanced reasoning)

    vSR-->>Gateway: Route to external model<br/>X-VSR-Model-Selected: bedrock/claude-sonnet<br/>X-VSR-Provider: aws-bedrock<br/>X-VSR-Category: advanced_reasoning

    Note over Client,Prometheus: Phase 3: Egress to External Provider

    Gateway->>Gateway: Resolve egress cluster for aws-bedrock<br/>Inject AWS credentials (SigV4 signing)
    Gateway->>Bedrock: POST https://bedrock-runtime.us-east-1.amazonaws.com<br/>/model/anthropic.claude-sonnet/invoke<br/>Authorization: AWS SigV4
    Bedrock-->>Gateway: Model response<br/>usage.input_tokens: 50, usage.output_tokens: 1450

    vSR->>vSR: Response filter: normalize token usage<br/>Map provider-specific fields to OpenAI format

    Gateway-->>Client: OpenAI-compatible response<br/>usage.total_tokens: 1500

    Note over Prometheus: Async metric emission (same metrics, external labels)
    Gateway->>Prometheus: vsr_requests_total{model="bedrock/claude-sonnet", provider="aws-bedrock", tier="premium"}
    Gateway->>Prometheus: vsr_tokens_consumed_total{model="bedrock/claude-sonnet", provider="aws-bedrock", tier="premium"} += 1500
```

### 2.3 Phase-by-Phase Detail

#### Phase 1: Authentication & QoS Protection

Uses existing RHCL (Kuadrant/Authorino/Limitador) with standard configuration. **No code changes to RHCL components.**

**AuthPolicy** -- vSR Access Authorization:

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: vsr-access-policy
  namespace: vsr-system
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: vsr-route
  rules:
    authentication:
      sa-token:
        kubernetesTokenReview:
          audiences:
            - vsr-api
    metadata:
      user-tier:
        http:
          url: "http://maas-api.redhat-ods-applications.svc:8080/api/v1/tier"
          method: GET
          headers:
            Authorization:
              selector: auth.identity.token
          credentials:
            authorizationHeader:
              prefix: Bearer
    response:
      success:
        headers:
          x-user-id:
            plain:
              selector: auth.identity.username
          x-tier:
            plain:
              selector: auth.metadata.user-tier.tier
```

This is the same pattern MaaS uses today. The only difference is the `targetRef` points to the vSR HTTPRoute instead of a model-specific route.

**RateLimitPolicy** -- QoS / DDoS Protection:

```yaml
apiVersion: kuadrant.io/v1
kind: RateLimitPolicy
metadata:
  name: vsr-qos-rate-limit
  namespace: vsr-system
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: vsr-route
  limits:
    global-qos:
      rates:
        - limit: 100
          window: 1m
      counters:
        - expression: auth.identity.username
    tier-free:
      rates:
        - limit: 10
          window: 1m
      counters:
        - expression: auth.identity.username
      when:
        - predicate: auth.metadata.user-tier.tier == 'free'
    tier-premium:
      rates:
        - limit: 60
          window: 1m
      counters:
        - expression: auth.identity.username
      when:
        - predicate: auth.metadata.user-tier.tier == 'premium'
    tier-enterprise:
      rates:
        - limit: 300
          window: 1m
      counters:
        - expression: auth.identity.username
      when:
        - predicate: auth.metadata.user-tier.tier == 'enterprise'
```

**Key design choice**: Rate limits are **request-count-based only** (not token-based). This is intentional for Phase 1:
- Token-based rate limiting requires response-body parsing (TokenRateLimitPolicy) which adds complexity
- QoS/DDoS protection is effectively achieved by capping request frequency
- Token consumption is captured via Prometheus for visibility, but not enforced at the gateway

#### Phase 2: Semantic Routing (vSR)

vSR operates as it does today with **no constraints from MaaS**. It receives the authenticated request with `X-User-ID` and `X-Tier` headers (for cache namespacing and observability) but no model lists or quotas.

**What vSR does (unchanged from standalone operation):**
1. PII detection and redaction
2. Jailbreak detection (blocks malicious prompts with 403)
3. Semantic classification via ModernBERT (domain, intent, complexity)
4. Semantic cache lookup (namespaced by `X-User-ID` for multi-tenant isolation)
5. Model selection based on classification result and configured routing rules

**What vSR adds for Phase 1 (new):**
- Injects `X-VSR-Model-Selected` header with the chosen model name
- Injects `X-VSR-Category` header with the classification result
- These headers are used downstream for Prometheus metric labeling only

**Multi-tenant cache isolation:**

Even in Phase 1, the semantic cache must be namespaced by user to prevent cross-tenant data leakage. vSR uses `X-User-ID` to scope cache entries:

```
Cache key: hash(X-User-ID + prompt_embedding)
```

This ensures User A never receives cached routing decisions from User B's similar prompts.

#### Phase 3: Execution & Observability

**Model Execution** -- KServe handles inference as it does today. No changes.

**Prometheus Metric Capture** -- The new component in Phase 1. After the response returns from KServe, metrics are emitted capturing the routing decision and token consumption.

### 2.4 Request Flow Headers

```http
# Client Request
POST /v1/chat/completions
Authorization: Bearer sa-token-xyz
Content-Type: application/json
{"messages": [{"role": "user", "content": "Solve this calculus problem..."}]}

# After Phase 1: Auth + QoS (✅ EXISTING headers)
POST /v1/chat/completions
Authorization: Bearer sa-token-xyz
X-User-ID: math-user-123             # ✅ User identity (from Authorino)
X-Tier: premium                      # ✅ Subscription tier (from Authorino)

# After Phase 2: vSR Semantic Routing (no model constraints injected)
# Example A: Internal model selected
POST /v1/chat/completions
Authorization: Bearer sa-token-xyz
X-User-ID: math-user-123             # ✅ Passed through
X-Tier: premium                      # ✅ Passed through
X-VSR-Model-Selected: llama3-70b     # 🆕 vSR's unconstrained model choice
X-VSR-Provider: kserve               # 🆕 Internal provider
X-VSR-Category: mathematics          # 🆕 Semantic classification result
Host: llama3-70b-service             # 🆕 Routing target (set by vSR ExtProc)

# Example B: External model selected
POST /v1/chat/completions
Authorization: Bearer sa-token-xyz
X-User-ID: math-user-123             # ✅ Passed through
X-Tier: premium                      # ✅ Passed through
X-VSR-Model-Selected: bedrock/claude-sonnet  # 🆕 External model choice
X-VSR-Provider: aws-bedrock          # 🆕 External provider
X-VSR-Category: advanced_reasoning   # 🆕 Semantic classification result
# Host/routing handled internally by vSR (egress to Bedrock endpoint)

# Phase 3: Response (after model execution)
HTTP/1.1 200 OK
Content-Type: application/json
X-VSR-Model-Selected: llama3-70b     # Passed through for client visibility
{"choices": [...], "usage": {"prompt_tokens": 50, "completion_tokens": 1450, "total_tokens": 1500}}
```

**Headers NOT present in Phase 1** (deferred to Phase 2+):
- ~~`X-Accessible-Models`~~ -- No model list injection
- ~~`X-Model-Quotas`~~ -- No per-model quota information
- ~~`X-MaaS-Model-Selected`~~ -- Replaced by `X-VSR-Model-Selected` (vSR decides, not MaaS)
- ~~`X-Tokens-Estimated`~~ -- No pre-estimation needed when there are no quota constraints
- ~~`X-MaaS-Quota-Remaining`~~ -- No quota enforcement in Phase 1

---

## 3. Egress & External Model Architecture

### 3.1 Overview

Phase 1 supports routing to external LLM providers alongside internal KServe models. From the client's perspective, the experience is identical -- the same `/v1/chat/completions` endpoint, the same auth flow, the same OpenAI-compatible response format. vSR decides whether an internal or external model is the best fit based on semantic classification.

**Supported external providers:**

| Provider | Authentication | Endpoint Pattern |
|----------|---------------|-----------------|
| AWS Bedrock | IAM (SigV4) or Access Keys | `bedrock-runtime.{region}.amazonaws.com` |
| Azure OpenAI | API Key or Azure AD | `{resource}.openai.azure.com` |
| Google Vertex AI | Service Account (OAuth2) | `{region}-aiplatform.googleapis.com` |
| OpenAI | API Key | `api.openai.com` |
| Anthropic | API Key | `api.anthropic.com` |

### 3.2 Two Egress Models: Gateway-Native vs. vSR-Managed

There is an active [OpenShift Gateway Egress Inference proposal](https://docs.google.com/document/d/1...) (Feb 2026, Foster/Rampal/Utt) that builds egress connectivity, auth token management, and API translation directly into the OpenShift Gateway (Istio/Envoy). This proposal is complementary to vSR -- it handles **infrastructure plumbing** while vSR handles **routing intelligence**.

Phase 1 is designed to work with either egress model, and to transition seamlessly from one to the other.

#### Separation of Concerns

| Responsibility | vSR (Intelligence) | Gateway (Plumbing) |
|----------------|--------------------|--------------------|
| Semantic classification | **vSR owns** | -- |
| Model selection decision | **vSR owns** | -- |
| PII detection / jailbreak | **vSR owns** | -- |
| Semantic caching | **vSR owns** | -- |
| Connectivity & trust to external services | -- | **Gateway owns** |
| Auth token injection (SigV4, API keys) | -- | **Gateway owns** |
| Inference API translation (OpenAI -> Bedrock) | -- | **Gateway owns** |
| Body-based routing (model name extraction) | -- | **Gateway owns** |
| TLS termination and certificate management | -- | **Gateway owns** |

**vSR's role is simple**: classify the request, select the best model, set a routing header (`X-VSR-Model-Selected`). It does not need to know how to authenticate to Bedrock or translate request formats. The gateway handles all of that.

#### Option A: Gateway-Native Egress (Preferred -- aligns with OpenShift Gateway proposal)

```mermaid
graph TB
    subgraph "OpenShift Cluster"
        subgraph "Gateway Layer"
            Gateway[maas-default-gateway]
            EgressGW[Egress Inference Gateway<br/>Connectivity + Trust + Auth + Translation]
            ATM[AuthTokenManagement<br/>Credential injection per provider]
            APITranslation[Inference API Translation<br/>WASM plugin: OpenAI -> provider-native]
        end

        subgraph "vsr-system"
            vSR[vSR ExtProc<br/>Classification + Selection only]
        end

        subgraph "Internal Models"
            KServe1[llama3-70b]
            KServe2[llama3-8b]
        end
    end

    subgraph "External Providers"
        Bedrock[AWS Bedrock]
        AzureOAI[Azure OpenAI]
        OpenAI[OpenAI API]
        Anthropic[Anthropic API]
    end

    Gateway --> vSR
    vSR -->|X-VSR-Model-Selected: llama3-70b| Gateway
    vSR -->|X-VSR-Model-Selected: bedrock/claude-sonnet| Gateway

    Gateway -->|Internal route| KServe1
    Gateway -->|Internal route| KServe2

    Gateway --> EgressGW
    EgressGW --> ATM
    ATM --> APITranslation
    EgressGW -->|TLS + SigV4| Bedrock
    EgressGW -->|TLS + API Key| AzureOAI
    EgressGW -->|TLS + API Key| OpenAI
    EgressGW -->|TLS + API Key| Anthropic
```

**How it works:**

1. vSR classifies the request and sets `X-VSR-Model-Selected: bedrock/claude-sonnet`
2. The gateway's HTTPRoute matches the model header and routes to the appropriate egress backend
3. The **AuthTokenManagement** (ATM) component (Authorino/WASM) matches the destination and injects provider credentials (SigV4, API key, etc.) into the request
4. The **Inference API Translation** layer (WASM plugin) converts the OpenAI-format request body to the provider's native format (e.g., Bedrock's Invoke API)
5. The egress gateway sends the request over TLS to the external provider
6. The response flows back through the translation layer (provider-native -> OpenAI format) and returns to the client

**What vSR does NOT need to do in this model:**
- No credential management (no Secrets mounted in vSR pod)
- No AWS SigV4 signing
- No request/response format translation
- No NetworkPolicy or ServiceEntry configuration
- No provider-specific adapters

**What vSR DOES do:**
- Semantic classification and model selection (its core value)
- Sets `X-VSR-Model-Selected` and `X-VSR-Provider` headers
- PII detection and redaction (before the request reaches the gateway egress)
- Prometheus metric emission (model, provider, category, tokens)

#### Option B: vSR-Managed Egress (Fallback -- if Gateway egress proposal is not ready)

If the OpenShift Gateway egress proposal has not landed yet, vSR can handle egress directly. This is the same architecture from the previous version of this document:

```mermaid
graph TB
    subgraph "OpenShift Cluster"
        subgraph "vsr-system"
            vSR[vSR Router<br/>ExtProc + Envoy<br/>+ Provider Adapters]
            Secrets[Kubernetes Secrets<br/>Provider Credentials]
        end

        subgraph "Egress Control"
            NP[NetworkPolicy<br/>Allow HTTPS egress]
            SE[ServiceEntry<br/>Per-provider endpoint]
        end

        subgraph "Internal Models"
            KServe1[llama3-70b]
            KServe2[llama3-8b]
        end
    end

    subgraph "External Providers"
        Bedrock[AWS Bedrock]
        AzureOAI[Azure OpenAI]
        OpenAI[OpenAI API]
        Anthropic[Anthropic API]
    end

    vSR -->|Internal route| KServe1
    vSR -->|Internal route| KServe2
    Secrets -.->|Mount/inject| vSR
    vSR -->|Egress + SigV4| Bedrock
    vSR -->|Egress + API Key| AzureOAI
    vSR -->|Egress + API Key| OpenAI
    vSR -->|Egress + API Key| Anthropic
    SE -.->|DNS resolution| vSR
    NP -.->|Allow egress| vSR
```

In this model, vSR takes on additional responsibilities:
- Provider credential management (K8s Secrets mounted in pod)
- AWS SigV4 signing (or delegates to AWS SDK)
- Request translation (OpenAI -> provider-native)
- Response normalization (provider-native -> OpenAI)
- vSR already has adapters for OpenAI and Anthropic in `pkg/openai/` and `pkg/anthropic/`; Bedrock and Vertex AI adapters would need to be added

### 3.3 Comparison: Gateway-Native vs. vSR-Managed Egress

| Aspect | Option A: Gateway-Native (Preferred) | Option B: vSR-Managed (Fallback) |
|--------|--------------------------------------|----------------------------------|
| vSR complexity | **Minimal** -- classification + headers only | Higher -- adapters, credential mgmt, signing |
| Credential security | Gateway manages credentials (Authorino/ATM) | vSR pod mounts Secrets directly |
| API translation | Gateway WASM plugin (shared infra) | vSR Go code (per-adapter) |
| New providers | Gateway team adds support once, all consumers benefit | vSR team must add adapter per provider |
| Dependency | Requires OpenShift Gateway egress proposal to land | Works today with existing vSR capabilities |
| Separation of concerns | Clean: vSR = intelligence, gateway = plumbing | Blurred: vSR does both intelligence and plumbing |
| Reusability | Gateway egress benefits all OpenShift workloads, not just vSR | Egress logic is vSR-specific |

**Recommendation**: Start with **Option A** (gateway-native) as the target architecture. If the gateway egress proposal timeline slips, **Option B** is a proven fallback -- vSR already has the adapter infrastructure. The transition from B to A is straightforward: remove provider adapters/secrets from vSR, configure gateway egress, and vSR continues setting the same routing headers.

### 3.4 Gateway Egress Configuration (Option A)

When using the gateway-native egress model, the following resources are needed. These are owned by the gateway/platform team, not the vSR team:

**Connectivity (Istio ServiceEntry + DestinationRule):**

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: aws-bedrock
spec:
  hosts:
    - "bedrock-runtime.us-east-1.amazonaws.com"
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
---
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: openai-api
spec:
  hosts:
    - "api.openai.com"
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
```

**Auth Token Management (new API from gateway proposal):**

```yaml
# Conceptual -- exact API TBD by gateway proposal
apiVersion: gateway.openshift.io/v1alpha1
kind: AuthTokenManagement
metadata:
  name: bedrock-auth
spec:
  targetRef:
    kind: ServiceEntry
    name: aws-bedrock
  authentication:
    type: aws-sigv4
    secretRef:
      name: aws-bedrock-credentials
    # Or IRSA:
    serviceAccountRef:
      name: gateway-egress-sa
      roleArn: "arn:aws:iam::123456789012:role/bedrock-access"
---
apiVersion: gateway.openshift.io/v1alpha1
kind: AuthTokenManagement
metadata:
  name: openai-auth
spec:
  targetRef:
    kind: ServiceEntry
    name: openai-api
  authentication:
    type: api-key
    secretRef:
      name: openai-credentials
    headerName: "Authorization"
    headerPrefix: "Bearer "
```

**Inference API Translation (gateway WASM plugin):**

The gateway proposal includes a WASM plugin for translating OpenAI Chat Completions format to provider-native formats. This handles:

| Direction | Translation |
|-----------|------------|
| Request: OpenAI -> Bedrock | `POST /v1/chat/completions` -> `POST /model/{id}/invoke` + Bedrock body format |
| Request: OpenAI -> Anthropic | `POST /v1/chat/completions` -> `POST /v1/messages` + Anthropic body format |
| Response: Bedrock -> OpenAI | `inputTokenCount`/`outputTokenCount` -> `usage.prompt_tokens`/`usage.completion_tokens` |
| Response: Anthropic -> OpenAI | `usage.input_tokens`/`usage.output_tokens` -> `usage.prompt_tokens`/`usage.completion_tokens` |

### 3.5 vSR-Managed Egress Configuration (Option B Fallback)

If the gateway egress proposal is not available, vSR handles egress directly. This requires additional configuration in the vSR namespace:

**Provider Credentials (K8s Secrets):**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vsr-aws-bedrock-credentials
  namespace: vsr-system
type: Opaque
data:
  AWS_ACCESS_KEY_ID: <base64>
  AWS_SECRET_ACCESS_KEY: <base64>
  AWS_REGION: dXMtZWFzdC0x  # us-east-1
---
apiVersion: v1
kind: Secret
metadata:
  name: vsr-openai-credentials
  namespace: vsr-system
type: Opaque
data:
  OPENAI_API_KEY: <base64>
```

**AWS IRSA (preferred over long-lived keys):**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vsr-router
  namespace: vsr-system
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789012:role/vsr-bedrock-access"
```

**Network Egress (NetworkPolicy):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: vsr-egress-allow-providers
  namespace: vsr-system
spec:
  podSelector:
    matchLabels:
      app: vsr-router
  policyTypes:
    - Egress
  egress:
    - to: []
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
    - to:
        - namespaceSelector: {}
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except: [10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16]
      ports:
        - { protocol: TCP, port: 443 }
```

### 3.6 External Model Routing Decision

Regardless of which egress model is used (gateway-native or vSR-managed), the routing decision is the same. vSR's semantic classification determines whether to route internally or externally based on the `config.yaml` model pool:

```yaml
# vSR config.yaml -- Internal + External model pool
models:
  domains:
    mathematics:
      model: llama3-70b                    # Internal KServe
      reasoning: true
      plugins: [system_prompt, semantic-cache]
    advanced_reasoning:
      model: bedrock/claude-sonnet         # External AWS Bedrock
      reasoning: true
      plugins: [system_prompt, semantic-cache]
    coding:
      model: granite-code-34b             # Internal KServe
      plugins: [system_prompt, semantic-cache]
    creative_writing:
      model: openai/gpt-4o                # External OpenAI
      plugins: [system_prompt, semantic-cache]
    general:
      model: llama3-8b                    # Internal KServe
      plugins: [semantic-cache]

endpoints:
  # Internal (KServe cluster services)
  llama3-70b:
    type: internal
    url: "llama3-70b.model-serving.svc.cluster.local"
  llama3-8b:
    type: internal
    url: "llama3-8b.model-serving.svc.cluster.local"
  granite-code-34b:
    type: internal
    url: "granite-code-34b.model-serving.svc.cluster.local"

  # External (cloud providers)
  bedrock/claude-sonnet:
    type: external
    provider: aws-bedrock
    model_id: "anthropic.claude-3-5-sonnet-20241022-v2:0"
    # Option A (gateway-native): no credentials_secret needed here
    # Option B (vSR-managed): credentials_secret: vsr-aws-bedrock-credentials
  openai/gpt-4o:
    type: external
    provider: openai
    model_id: "gpt-4o"
    # Option A: no credentials_secret needed
    # Option B: credentials_secret: vsr-openai-credentials
```

**Routing criteria**: vSR selects between internal and external models purely based on semantic fit. The `config.yaml` defines which domains map to which models, and the operator configures this at deployment time. The key difference between egress options is only who handles credentials and translation -- not who makes the routing decision.

### 3.7 Transition Path: Option B -> Option A

When the OpenShift Gateway egress proposal lands, transitioning from vSR-managed to gateway-native egress requires:

1. **Deploy gateway egress resources**: ServiceEntry, AuthTokenManagement, API translation WASM plugin
2. **Remove from vSR**: Provider credential Secrets, NetworkPolicy, adapter code for external providers
3. **Update vSR config**: Remove `credentials_secret` from external endpoint configs
4. **No changes to**: vSR classification, model selection, header injection, Prometheus metrics, client API

vSR continues setting `X-VSR-Model-Selected: bedrock/claude-sonnet` exactly as before. The only change is that the gateway now handles the downstream plumbing instead of vSR's internal adapters.

---

## 4. Observability Architecture

### 3.1 Prometheus Metrics

Phase 1 introduces metrics that capture vSR routing decisions and model token consumption. These metrics serve dual purposes: operational monitoring today, and billing/quota data source for future phases.

#### Metric Definitions

```yaml
# Counter: Total requests processed by vSR
- name: vsr_requests_total
  type: counter
  labels: [user_id, tier, category, model_selected, provider, status]
  description: "Total requests routed through vSR"
  # provider: "kserve", "aws-bedrock", "azure-openai", "openai", "anthropic", "vertex-ai"

# Counter: Total tokens consumed across all vSR requests
- name: vsr_tokens_consumed_total
  type: counter
  labels: [user_id, tier, model_selected, provider, token_type]
  description: "Total tokens consumed (prompt + completion)"
  # token_type: "prompt", "completion", "total"
  # provider: "kserve", "aws-bedrock", "azure-openai", "openai", "anthropic", "vertex-ai"

# Histogram: vSR classification latency
- name: vsr_classification_duration_seconds
  type: histogram
  labels: [category]
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25]
  description: "Time spent in semantic classification"

# Histogram: End-to-end request latency through vSR
- name: vsr_request_duration_seconds
  type: histogram
  labels: [tier, model_selected, cache_hit]
  buckets: [0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30]
  description: "Total request duration including model inference"

# Counter: Security events (PII detections, jailbreak blocks)
- name: vsr_security_events_total
  type: counter
  labels: [user_id, tier, event_type]
  description: "Security events detected by vSR"
  # event_type: "pii_detected", "pii_redacted", "jailbreak_blocked"

# Counter: Semantic cache performance
- name: vsr_cache_operations_total
  type: counter
  labels: [tier, operation]
  description: "Semantic cache hits and misses"
  # operation: "hit", "miss"

# Gauge: Models currently available in vSR routing pool
- name: vsr_available_models
  type: gauge
  labels: [model, provider]
  description: "Models available for routing (internal and external)"

# Counter: External provider egress requests
- name: vsr_external_requests_total
  type: counter
  labels: [provider, model_selected, status]
  description: "Requests sent to external providers via egress"
  # status: "success", "error", "timeout"

# Histogram: External provider response latency
- name: vsr_external_latency_seconds
  type: histogram
  labels: [provider, model_selected]
  buckets: [0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30, 60]
  description: "Latency for external provider responses (includes network egress)"
```

#### Where Metrics Are Emitted

```mermaid
graph LR
    subgraph "Metric Sources"
        vSR[vSR ExtProc<br/>classification_duration<br/>security_events<br/>cache_operations<br/>model_selected]
        Envoy[Envoy/Gateway<br/>request_duration<br/>requests_total]
        KServe[KServe Response<br/>tokens_consumed via<br/>usage.total_tokens parsing]
    end

    subgraph "Collection"
        Prom[Prometheus<br/>:9190 scrape from vSR<br/>:15090 scrape from Envoy]
    end

    subgraph "Visualization"
        Grafana[Grafana Dashboards]
    end

    vSR --> Prom
    Envoy --> Prom
    KServe --> Prom
    Prom --> Grafana
```

**Token capture approach**: vSR's ExtProc response filter parses the `usage` object from the OpenAI-compatible response body. This is already supported by vSR's response filter pipeline -- no new parsing logic is needed. The token counts are emitted as Prometheus counter increments labeled by model and user.

### 3.2 Grafana Dashboard Panels (Recommended)

| Panel | Metric Source | Purpose |
|-------|--------------|---------|
| Requests per model (timeseries) | `vsr_requests_total` | See which models vSR is routing to |
| Tokens per model per tier (timeseries) | `vsr_tokens_consumed_total` | Understand cost distribution before billing exists |
| Classification distribution (pie chart) | `vsr_requests_total` by `category` | Validate semantic classification accuracy |
| Cache hit ratio (gauge) | `vsr_cache_operations_total` | Measure cache effectiveness |
| Security events (timeseries) | `vsr_security_events_total` | Monitor PII and jailbreak detections |
| P50/P95/P99 latency (heatmap) | `vsr_request_duration_seconds` | End-to-end performance monitoring |
| Rate limit rejections (timeseries) | Limitador metrics | Monitor QoS policy effectiveness |
| Per-user token usage (table) | `vsr_tokens_consumed_total` | Identify heavy users (pre-billing insight) |
| Internal vs external routing (pie) | `vsr_requests_total` by `provider` | See traffic split between KServe and external providers |
| External provider latency (heatmap) | `vsr_external_latency_seconds` | Monitor egress performance per provider |
| External provider errors (timeseries) | `vsr_external_requests_total{status="error"}` | Detect provider outages or credential issues |
| Token cost by provider (table) | `vsr_tokens_consumed_total` by `provider` | Pre-billing cost estimation across providers |

---

## 4. Implementation Requirements

### 4.1 Component Change Summary

#### RHCL (Kuadrant / Authorino / Limitador) -- No Code Changes

| Component | Change | Status |
|-----------|--------|--------|
| Authorino | AuthPolicy targeting vSR HTTPRoute | ✅ Configuration only |
| Limitador | RateLimitPolicy for QoS (request-count based) | ✅ Configuration only |
| Kuadrant Operator | No changes | ✅ Existing |
| WASM Shim | No changes | ✅ Existing |

The RHCL stack is deployed and configured exactly as it is for MaaS today. The only difference is the policy `targetRef` points to the vSR HTTPRoute.

#### MaaS API -- No Code Changes

| Component | Change | Status |
|-----------|--------|--------|
| Tier resolution endpoint | Already exists, vSR reuses it via Authorino metadata | ✅ Existing |
| Token management | Existing SA token + API key infrastructure | ✅ Existing |
| Model discovery | Not needed in Phase 1 (vSR has its own model pool config) | Deferred |
| Accessible-models API | Not needed in Phase 1 | Deferred |
| Usage tracking API | Not needed in Phase 1 (Prometheus replaces this) | Deferred |

#### vSR (Semantic Router) -- Minimal Changes

| Component | Change | Option A (GW Egress) | Option B (vSR Egress) |
|-----------|--------|---------------------|----------------------|
| ExtProc service | No changes to core routing logic | ✅ Existing | ✅ Existing |
| PII / Jailbreak detection | No changes | ✅ Existing | ✅ Existing |
| Semantic classification | No changes | ✅ Existing | ✅ Existing |
| External provider adapters | Request/response translation | **Not needed** | 🆕 Enhancement |
| AWS SigV4 signing | Credential handling for Bedrock | **Not needed** | 🆕 New |
| Semantic cache | Namespace by `X-User-ID` | 🆕 Enhancement | 🆕 Enhancement |
| `X-VSR-Model-Selected` header | Inject model name | 🆕 Enhancement | 🆕 Enhancement |
| `X-VSR-Provider` header | Inject provider type | 🆕 New | 🆕 New |
| `X-VSR-Category` header | Inject classification result | 🆕 Enhancement | 🆕 Enhancement |
| Prometheus metrics | Routing + token metrics with provider labels | 🆕 New | 🆕 New |

With **Option A (Gateway-native egress)**, vSR's scope is strictly classification, model selection, and header injection. No provider-specific code is needed in vSR.

#### Gateway / Envoy

| Component | Change | Option A (GW Egress) | Option B (vSR Egress) |
|-----------|--------|---------------------|----------------------|
| HTTPRoute for vSR | Route to vSR service | 🆕 Configuration | 🆕 Configuration |
| ExtProc filter | Envoy calls vSR ExtProc | 🆕 Configuration | 🆕 Configuration |
| Header sanitization | Strip `X-VSR-*` from client requests | 🆕 Configuration | 🆕 Configuration |
| ServiceEntry (per provider) | Declare external hosts | 🆕 Gateway team owns | 🆕 vSR team configures |
| AuthTokenManagement | Credential injection per provider | 🆕 Gateway team owns | **Not needed** |
| API Translation WASM | OpenAI -> provider-native | 🆕 Gateway team owns | **Not needed** |
| Egress routing (HTTPRoute) | Route by `X-VSR-Model-Selected` to external backends | 🆕 Gateway team owns | **Not needed** |

#### Egress & External Providers (Option B only)

These resources are only needed if using vSR-managed egress (Option B fallback):

| Component | Change | Status |
|-----------|--------|--------|
| Provider credential Secrets | K8s Secrets for AWS, Azure, OpenAI, Anthropic keys | 🆕 Configuration |
| AWS IRSA (if on ROSA/EKS) | ServiceAccount annotation for IAM role binding | 🆕 Configuration |
| NetworkPolicy (egress) | Allow vSR pods to reach external HTTPS endpoints | 🆕 Configuration |
| vSR model pool config | Add `credentials_secret` to external endpoint entries | 🆕 Configuration |

### 4.2 Gateway Configuration

#### HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: vsr-route
  namespace: vsr-system
spec:
  parentRefs:
    - name: maas-default-gateway
      namespace: openshift-ingress
  hostnames:
    - "vsr.${CLUSTER_DOMAIN}"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1/chat/completions
      backendRefs:
        - name: vsr-envoy
          port: 8888
```

#### Header Sanitization

The gateway must strip any `X-VSR-*` headers from incoming requests to prevent clients from spoofing routing metadata:

```yaml
# Envoy route config filter
request_headers_to_remove:
  - "x-vsr-model-selected"
  - "x-vsr-category"
  - "x-user-id"
  - "x-tier"
```

### 4.3 Kubernetes Resources

**Option A: Gateway-native egress (preferred)**
```
vsr-system namespace:                       # vSR team owns
├── Deployment: vsr-router                  # vSR ExtProc + Envoy sidecar
├── Service: vsr-envoy                      # ClusterIP, port 8888
├── Service: vsr-extproc                    # ClusterIP, port 50051
├── Service: vsr-metrics                    # ClusterIP, port 9190 (Prometheus scrape)
├── ServiceMonitor: vsr-metrics             # Prometheus ServiceMonitor
├── ConfigMap: vsr-config                   # vSR routing config (config.yaml)
├── HTTPRoute: vsr-route                    # Gateway API route
├── AuthPolicy: vsr-access-policy           # Kuadrant auth policy
└── RateLimitPolicy: vsr-qos-rate-limit     # Kuadrant rate limit policy

gateway namespace:                          # Gateway/platform team owns
├── ServiceEntry: aws-bedrock               # External host declaration
├── ServiceEntry: openai-api                # External host declaration
├── ServiceEntry: anthropic-api             # External host declaration
├── AuthTokenManagement: bedrock-auth       # AWS SigV4 credential injection
├── AuthTokenManagement: openai-auth        # API key injection
├── AuthTokenManagement: anthropic-auth     # API key injection
├── Secret: aws-bedrock-credentials         # Provider credentials
├── Secret: openai-credentials              # Provider credentials
├── Secret: anthropic-credentials           # Provider credentials
└── WasmPlugin: api-translation             # OpenAI -> provider-native translation
```

**Option B: vSR-managed egress (fallback)**
```
vsr-system namespace:                       # vSR team owns everything
├── Deployment: vsr-router                  # vSR ExtProc + Envoy sidecar
├── ServiceAccount: vsr-router              # With IRSA annotation for AWS (if applicable)
├── Service: vsr-envoy                      # ClusterIP, port 8888
├── Service: vsr-extproc                    # ClusterIP, port 50051
├── Service: vsr-metrics                    # ClusterIP, port 9190 (Prometheus scrape)
├── ServiceMonitor: vsr-metrics             # Prometheus ServiceMonitor
├── ConfigMap: vsr-config                   # vSR routing config (config.yaml)
├── Secret: vsr-aws-bedrock-credentials     # AWS Bedrock credentials
├── Secret: vsr-openai-credentials          # OpenAI API key
├── Secret: vsr-anthropic-credentials       # Anthropic API key
├── NetworkPolicy: vsr-egress-allow-providers  # Egress to external HTTPS endpoints
├── ServiceEntry: aws-bedrock               # (Istio only) External host declaration
├── ServiceEntry: openai-api                # (Istio only) External host declaration
├── ServiceEntry: anthropic-api             # (Istio only) External host declaration
├── HTTPRoute: vsr-route                    # Gateway API route
├── AuthPolicy: vsr-access-policy           # Kuadrant auth policy
└── RateLimitPolicy: vsr-qos-rate-limit     # Kuadrant rate limit policy
```

---

## 5. Error Handling

Phase 1 uses standard OpenAI-compatible error responses. Error handling is simpler than v1.0/v1.1 because there are no model constraints or fallback chains.

### 5.1 Error Responses

**Authentication Failure (Authorino rejects):**
```json
{
  "error": {
    "message": "Invalid authentication credentials",
    "type": "invalid_request_error",
    "code": "invalid_api_key"
  }
}
```
HTTP Status: `401 Unauthorized`

**Rate Limited (Limitador rejects):**
```json
{
  "error": {
    "message": "Rate limit exceeded. Please slow down.",
    "type": "rate_limit_error",
    "code": "rate_limit_exceeded"
  }
}
```
HTTP Status: `429 Too Many Requests`
Headers: `Retry-After: 60`

**Security Violation (vSR jailbreak detection):**
```json
{
  "error": {
    "message": "Request blocked for safety violations",
    "type": "policy_violation",
    "code": "content_policy_violation"
  }
}
```
HTTP Status: `403 Forbidden`

**Model Unavailable (KServe backend down):**
```json
{
  "error": {
    "message": "The selected model is currently unavailable. Please try again.",
    "type": "server_error",
    "code": "model_unavailable"
  }
}
```
HTTP Status: `503 Service Unavailable`

### 5.2 Error Flow

```mermaid
graph TD
    Request[Client Request]
    Auth{Authorino:<br/>Valid token?}
    RateLimit{Limitador:<br/>Within rate?}
    Security{vSR:<br/>Safe content?}
    Route[vSR: Route to model]
    Execute{KServe:<br/>Model healthy?}
    Response[200 OK + Response]

    Request --> Auth
    Auth -->|No| E401[401 Unauthorized]
    Auth -->|Yes| RateLimit
    RateLimit -->|No| E429[429 Too Many Requests]
    RateLimit -->|Yes| Security
    Security -->|Jailbreak| E403[403 Forbidden]
    Security -->|Safe| Route
    Route --> Execute
    Execute -->|Down| E503[503 Service Unavailable]
    Execute -->|OK| Response
```

**No fallback logic in Phase 1.** If the model vSR selects is unavailable, the request fails with 503. Intelligent fallback is deferred to Phase 2.

---

## 6. Security Considerations

### 6.1 Trust Boundaries

```
┌─────────────────────────────────────────────────────┐
│ UNTRUSTED: Client / External                        │
│  - SA token (validated by Authorino)                │
│  - Request body (scanned by vSR for PII/jailbreak)  │
└──────────────────────┬──────────────────────────────┘
                       │ Gateway strips X-VSR-*, X-User-ID, X-Tier
                       ▼
┌─────────────────────────────────────────────────────┐
│ TRUSTED: Internal (after Authorino validation)      │
│  - X-User-ID, X-Tier (set by Authorino)             │
│  - X-VSR-Model-Selected (set by vSR ExtProc)        │
│  - X-VSR-Category (set by vSR ExtProc)              │
└─────────────────────────────────────────────────────┘
```

### 6.2 Phase 1 Security Model

| Threat | Mitigation | Component |
|--------|-----------|-----------|
| Unauthenticated access | AuthPolicy validates SA token via Kubernetes TokenReview | Authorino |
| DDoS / request flooding | RateLimitPolicy caps request rate per user and tier | Limitador |
| Prompt injection / jailbreak | ModernBERT jailbreak classifier (96.7% accuracy) | vSR |
| PII in prompts | ModernBERT PII detector (95.7% F1) with redaction | vSR |
| Header spoofing | Gateway strips X-VSR-*, X-User-ID, X-Tier from incoming requests | Envoy |
| Cross-tenant cache leakage | Semantic cache namespaced by X-User-ID | vSR |
| Unauthorized model access | Not enforced in Phase 1 (vSR routes to its own configured pool) | Deferred to Phase 2 |
| Credential leakage (external providers) | **Option A**: Gateway manages credentials -- vSR pod has no provider secrets. **Option B**: Credentials in K8s Secrets with namespace RBAC; IRSA preferred for AWS | Gateway ATM / K8s Secrets |
| Uncontrolled egress | **Option A**: Gateway controls egress via ServiceEntry. **Option B**: NetworkPolicy restricts vSR pod egress | Gateway / NetworkPolicy |
| PII sent to external providers | PII detection and redaction runs in vSR before model routing -- applies equally to internal and external models | vSR |
| External provider MITM | All egress uses TLS; provider certificates validated by system trust store | Gateway / Envoy TLS |
| External provider data retention | Out of scope for Phase 1 -- governed by provider contracts (BAA, DPA) | Operational |

### 6.3 Egress Security Controls

**Option A (Gateway-native egress)** provides the strongest security posture because vSR never touches provider credentials:

```
┌─────────────────────────────────────────────────────────────┐
│ vSR Pod                                                     │
│  - NO provider credentials in pod                           │
│  - Only sets routing headers (X-VSR-Model-Selected)         │
│  - PII redacted before request leaves vSR                   │
└──────────────────────┬──────────────────────────────────────┘
                       │ X-VSR-Model-Selected: bedrock/claude-sonnet
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Gateway Egress (Istio/Envoy)                                │
│  - AuthTokenManagement injects SigV4/API key                │
│  - API Translation WASM converts request format             │
│  - ServiceEntry restricts allowed external hosts            │
│  - Credentials never exposed to application layer           │
└──────────────────────┬──────────────────────────────────────┘
                       │ TLS + provider auth
                       ▼
               External Providers
```

**Option B (vSR-managed egress)** requires additional security controls:

```
┌─────────────────────────────────────────────────────────────┐
│ vSR Pod                                                     │
│  - Mounts provider credential Secrets (read-only)           │
│  - IRSA ServiceAccount for AWS (no static keys in pod)      │
│  - Credentials never logged, never in headers to client     │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        │ NetworkPolicy / ServiceEntry │
        │ ALLOW:                       │
        │  - *.amazonaws.com:443       │
        │  - *.openai.azure.com:443    │
        │  - api.openai.com:443        │
        │  - api.anthropic.com:443     │
        │ DENY: everything else        │
        └──────────────┬──────────────┘
                       │ TLS only
                       ▼
               External Providers
```

**Credential rotation**: For Option A, the gateway team manages credential rotation independently of vSR. For Option B, Kubernetes Secrets support rotation. AWS IRSA credentials are automatically rotated by the token webhook. API key-based providers should be rotated periodically, and vSR should re-read credentials without restart (via Secret volume mount with `fsnotify` or periodic reload).

**PII protection for external routing**: This is critical when routing to external providers. The PII detection/redaction step in vSR runs before model selection, which means sensitive data is redacted before it ever leaves the cluster. This applies regardless of which egress model is used -- vSR's PII filter always runs first.

### 6.4 What Phase 1 Does NOT Protect Against

- **Unauthorized access to specific models**: Any authenticated user with vSR access can potentially be routed to any model in vSR's pool (internal or external). Per-model RBAC is deferred to Phase 2.
- **Token overconsumption**: No per-model or per-user token quotas are enforced. A user could consume large amounts of tokens within the request rate limit. Prometheus metrics provide visibility but not enforcement.
- **Cost-based abuse**: Without billing enforcement, a free-tier user could be routed to an expensive external model (e.g., GPT-4o via OpenAI). This is acceptable in Phase 1 because the vSR model pool should be curated to match the intended audience.
- **External provider cost overruns**: No budget caps on external API spend. Prometheus captures usage but does not enforce limits.

**Mitigation for Phase 1**: Operators should configure vSR's model pool (`config.yaml`) to only include models appropriate for the expected user base and budget. This is a deployment-time control, not a runtime policy. For external providers, operators should set provider-side spending limits (e.g., AWS Bedrock quotas, OpenAI usage limits) as a backstop.

---

## 7. Deployment Topology

### 7.1 Namespace Layout

```mermaid
graph TB
    subgraph "openshift-ingress"
        Gateway[maas-default-gateway]
    end

    subgraph "kuadrant-system"
        Authorino[Authorino]
        Limitador[Limitador]
    end

    subgraph "redhat-ods-applications"
        MaaSAPI[MaaS API<br/>Tier resolution only]
    end

    subgraph "vsr-system"
        vSRDeploy[vSR Router Deployment<br/>ExtProc + Envoy]
        vSRConfig[vSR Config<br/>config.yaml]
        AuthP[AuthPolicy]
        RLP[RateLimitPolicy]
        Route[HTTPRoute]
        SvcMon[ServiceMonitor]
    end

    subgraph "model-serving namespaces"
        KServe1[llama3-70b InferenceService]
        KServe2[llama3-8b InferenceService]
        KServe3[granite-code-34b InferenceService]
    end

    subgraph "Gateway Egress (Option A)"
        EgressGW[Egress Inference Gateway<br/>ATM + API Translation]
    end

    subgraph "External Providers"
        Bedrock[AWS Bedrock]
        AzureOAI[Azure OpenAI]
        OpenAIAPI[OpenAI API]
        AnthropicAPI[Anthropic API]
    end

    subgraph "monitoring"
        Prometheus[Prometheus]
        Grafana[Grafana]
    end

    Gateway --> Route
    Route --> vSRDeploy
    AuthP -.-> Authorino
    RLP -.-> Limitador
    Authorino -.-> MaaSAPI
    vSRDeploy -->|Internal| KServe1
    vSRDeploy -->|Internal| KServe2
    vSRDeploy -->|Internal| KServe3
    vSRDeploy -->|X-VSR-Model-Selected| Gateway
    Gateway --> EgressGW
    EgressGW -->|TLS + Auth| Bedrock
    EgressGW -->|TLS + Auth| AzureOAI
    EgressGW -->|TLS + Auth| OpenAIAPI
    EgressGW -->|TLS + Auth| AnthropicAPI
    SvcMon --> Prometheus
    Prometheus --> Grafana
```

### 7.2 vSR Model Pool Configuration

In Phase 1, vSR's model pool is configured statically in `config.yaml`. This replaces the dynamic `X-Accessible-Models` approach from v1.1. The configuration supports both internal (KServe) and external (cloud provider) models uniformly.

See Section 3.7 for the full configuration example including both internal and external endpoints.

**Operator responsibilities for external models:**
- Provision and configure provider credentials as Kubernetes Secrets
- Set provider-side spending limits as a backstop (e.g., AWS Bedrock service quotas, OpenAI usage limits)
- Configure NetworkPolicy or ServiceEntry for egress to each provider
- Choose appropriate models per domain based on cost, latency, and capability trade-offs

---

## 8. Phase 1 to Phase 2 Transition Path

Phase 1 is designed so that every component can be extended without rework when Phase 2 adds per-model authorization and quota enforcement.

| Phase 1 Component | Phase 2 Extension |
|-------------------|-------------------|
| AuthPolicy (vSR access) | Add per-model RBAC via MaaS API `accessible-models` endpoint |
| RateLimitPolicy (QoS only) | Add TokenRateLimitPolicy for per-model token quotas |
| Prometheus metrics (capture) | Metrics become input to billing system and quota enforcement |
| vSR static model pool | Replace with dynamic model list from MaaS API headers |
| No fallback logic | Add constrained re-routing when quota-limited models are exhausted |
| `X-VSR-Model-Selected` header | Becomes `X-MaaS-Model-Selected` after MaaS validates the selection |
| External model egress (Option A: gateway / Option B: vSR) | Converge on gateway-native egress; MaaS API manages external provider registry; dynamic provider discovery |
| Provider-side spending limits only | Per-provider budget enforcement via Limitador + MaaS billing engine |
| No cost differentiation | Cost-aware routing: vSR considers per-model cost when selecting (internal cheaper than external) |

**Data continuity**: The Prometheus metrics collected in Phase 1 (`vsr_tokens_consumed_total` by model, user, tier, provider) provide the historical data needed to configure accurate per-model token quotas and per-provider budgets when Phase 2 launches. This means Phase 1 is not throwaway -- it is the data collection period for Phase 2 policy tuning. The `provider` label on all metrics gives immediate visibility into internal vs. external cost distribution.

---

## 9. Conclusion

Phase 1 delivers a production-ready vSR integration with MaaS by making the minimum viable set of changes:

| Category | Scope |
|----------|-------|
| RHCL changes | **Zero** -- configuration only (AuthPolicy + RateLimitPolicy) |
| MaaS API changes | **Zero** -- reuses existing tier resolution endpoint |
| vSR changes | **Minimal** -- cache namespacing + header injection + provider adapters + Prometheus metrics |
| Gateway changes | **Configuration** -- HTTPRoute + header sanitization + ExtProc filter |
| Egress / External models | **Option A (preferred)**: Gateway team owns egress plumbing; vSR just sets routing headers. **Option B (fallback)**: vSR-managed adapters + Secrets + NetworkPolicy |

The architecture follows a deliberate strategy: **observe before you enforce**. By capturing model selection and token consumption in Prometheus (with `provider` labels distinguishing internal KServe from external AWS Bedrock, OpenAI, etc.) before building enforcement policies, Phase 1 provides real usage data to inform Phase 2's per-model quotas, per-provider budgets, and billing rates.

The egress architecture is designed around a clean separation of concerns: **vSR is the intelligence layer** (classification, model selection, security filtering) and the **OpenShift Gateway is the plumbing layer** (connectivity, credentials, API translation). The [OpenShift Gateway Egress Inference proposal](https://docs.google.com/document/d/1...) (Feb 2026) builds this plumbing directly into the platform, making vSR's job simpler -- it only needs to set a routing header, not manage provider credentials or translate API formats. If the gateway proposal timeline slips, vSR can handle egress directly as a fallback, with a clean transition path to gateway-native egress when it becomes available.

Phase 1 is not a prototype -- it is the first production layer. Every component deployed in Phase 1 remains in place as Phase 2 adds authorization depth and cost controls on top of it.
