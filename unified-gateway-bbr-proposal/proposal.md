# Proposal: Unified Gateway Flow via Pluggable BBR

**Date:** April 15, 2026
**Status:** Draft / RFC
**Tracker:** RHAIRFE-1304 (Unified Entry Point)

---

## Executive Summary

This proposal introduces an alternative approach to the unified entry point (RHAIRFE-1304) that decouples the gateway flow from the tight integration with RHCL/Kuadrant. Instead of adding a third Envoy filter to work around Kuadrant's limitations, we consolidate the entire request lifecycle — authentication, rate limiting, and payload processing — into a single BBR ext_proc pipeline with pluggable plugins.

The core idea: **BBR becomes the single orchestration layer**, with auth and rate-limiting as swappable plugins that call existing backends (Authorino, Limitador) via gRPC. This removes the Kuadrant operator and WasmPlugin from the critical path while keeping proven backend services and enabling future pluggability.

---

## Problem Statement

### The Unified Entry Point Challenge

Today, model inference requests include the **model name** and **namespace** in the URL path:

```
POST https://maas.example.com/ llm / granite-3b /v1/chat/completions
                                ^^^   ^^^^^^^^^^
                            namespace  model name
```

Industry standard (OpenAI-compatible) APIs use a single endpoint with the model in the **request body**:

```
POST https://maas.example.com/v1/chat/completions
Body: {"model": "granite-3b", "messages": [...]}
```

To align with industry standards (RHAIRFE-1304), we need to remove the model from the URL path.

### Why This Breaks the Current Architecture

The root cause is **filter ordering in Envoy**. Today, the Kuadrant WasmPlugin (auth + rate limiting) runs as the **first filter**, before BBR (payload processing). Kuadrant uses the model name from the URL path to determine which AuthPolicy and TokenRateLimitPolicy apply to the request. **Kuadrant's WasmPlugin cannot read the request body** — it only sees headers and the path. If the model is no longer in the path, Kuadrant has no way to identify which model the request targets, and per-model auth and rate-limiting break.

```mermaid
graph LR
    subgraph f1["Filter 1: Kuadrant WasmPlugin"]
        A1["Reads model from PATH<br/>Calls Authorino (auth)<br/>Calls Limitador (rate)<br/><br/>⚠ CANNOT read request body"]
    end
    subgraph f2["Filter 2: BBR ext_proc"]
        B1["body-field-to-header<br/>provider-resolver<br/>api-translation<br/>apikey-injection"]
    end
    f1 --> f2
```

> If model is removed from path → Kuadrant can't identify the model → auth/rate-limit break

### Why We Can't Simply Swap the Filter Order

A natural thought: swap BBR and Kuadrant so BBR runs first, extracts the model from the body, and Kuadrant can use it. **This doesn't work** because:

1. **api-translation replaces the user's API key** with the provider's API key. If Kuadrant runs after BBR, it sees the provider key (`sk-ant-...`) instead of the user key (`sk-oai-...`). Auth and rate limiting break — Kuadrant can't identify the user.

2. **Heavy payload processing should only run after auth passes.** api-translation, apikey-injection, and provider-resolver do significant work (external CRD lookups, request body transformation, secret access). Running these before auth means unauthenticated requests consume resources unnecessarily.

3. **Some payload plugins depend on auth data.** For example, subscription-based routing and metering need to know the user identity, which comes from the auth step.

### Why the Lightweight ext_proc Approach Ends Up with 3 Hops

The proposed workaround is to split BBR: put a lightweight ext_proc **before** Kuadrant that only does `body-field-to-header` (extract model from body → inject as header). Then Kuadrant can read the model from the header.

However, this results in **three separate Envoy filters**:

```
Filter 1: ext_proc (lightweight)    →  gRPC hop to lightweight BBR service
Filter 2: WasmPlugin (Kuadrant)     →  in-process WASM + gRPC to Authorino + gRPC to Limitador
Filter 3: ext_proc (BBR)            →  gRPC hop to BBR service (remaining plugins)
```

Three filters, two ext_proc services to deploy, and the tight Kuadrant coupling remains unchanged. This solves the immediate unified-entry-point problem but does not address the underlying architectural constraints.

### The Tight Coupling Problem

Beyond the unified entry point, the current architecture has three layers of CRD indirection:

```
MaaSAuthPolicy → MaaS Controller → Kuadrant AuthPolicy → Kuadrant Operator → Authorino AuthConfig
MaaSSubscription → MaaS Controller → Kuadrant TRLP → Kuadrant Operator → Limitador Rules
```

This coupling introduces:
- **Deployment complexity** — Kuadrant operator requires CSV patching for OpenShift Gateway controller recognition, race conditions with `MissingDependency` status
- **Architectural constraints** — WasmPlugin can't see request body, forcing model-in-path pattern
- **Limited pluggability** — auth and rate-limiting are locked to Authorino and Limitador via Kuadrant's CRDs; no path to swap auth backends or replace Limitador with a metering system
- **Multiple Envoy filters** — WasmPlugin + ext_proc adds latency and operational complexity
- **Duplicate body opening** — Limitador's TokenRateLimitPolicy opens the response body to extract token counts, while BBR also opens the response body for api-translation. Two separate components parsing the same body

---

## Architecture Comparison

### Side-by-Side Overview

#### Approach A: Current (2 filters, tight Kuadrant coupling)

```mermaid
graph LR
    subgraph f1["Filter 1: WasmPlugin — Kuadrant"]
        A["WASM binary (in-process)<br/>→ Authorino gRPC<br/>→ Limitador gRPC<br/><br/>⚠ Can't see body"]
    end
    subgraph f2["Filter 2: ext_proc — BBR"]
        B["body-field-to-header<br/>provider-resolver<br/>api-translation<br/>apikey-injection"]
    end
    f1 --> f2
```

> **⚠ Model MUST be in URL path.** Kuadrant reads the model from the path to match per-model policies.
> Per-model HTTPRoutes required for policy targeting.

#### Approach B: Lightweight ext_proc before Kuadrant (3 filters)

```mermaid
graph LR
    subgraph f1["Filter 1: ext_proc — lightweight"]
        A["body-field-to-header<br/>(model → header)"]
    end
    subgraph f2["Filter 2: WasmPlugin — Kuadrant"]
        B["WASM binary (in-process)<br/>→ Authorino gRPC<br/>→ Limitador gRPC"]
    end
    subgraph f3["Filter 3: ext_proc — BBR"]
        C["provider-resolver<br/>api-translation<br/>apikey-injection"]
    end
    f1 --> f2 --> f3
```

> **⚠ 3 filters, 2 ext_proc services to deploy, 2 gRPC hops.** Kuadrant coupling remains unchanged.

#### Approach C: Unified BBR Pipeline (this proposal — 1 filter, pluggable)

```mermaid
graph LR
    subgraph f1["Single Filter: ext_proc — BBR"]
        A["1. body-field-to-header<br/>2. auth-plugin → Authorino gRPC ← <i>swappable</i><br/>3. rate-limit-plugin → Limitador gRPC ← <i>swappable</i><br/>4. model-provider-resolver<br/>5. api-translation<br/>6. apikey-injection"]
    end
```

> **✅ Single filter, single gRPC hop.** No Kuadrant dependency.
> Auth + rate-limit as swappable BBR plugins. Model read from body natively.

---

### Approach A: Current Architecture (Today)

Two Envoy filters in the gateway filter chain:

1. **Kuadrant WasmPlugin** (runs in-process in Envoy)
   - Calls Authorino via gRPC for authentication/authorization
   - Calls Limitador via gRPC for rate limiting
2. **BBR ext_proc** (external gRPC service)
   - body-field-to-header, provider-resolver, api-translation, apikey-injection

```mermaid
graph LR
    Client([Client]) --> Envoy

    subgraph Envoy["Envoy Gateway"]
        direction TB
        WP["Filter 1: Kuadrant WasmPlugin<br/>(in-process WASM)"]
        EP["Filter 2: BBR ext_proc<br/>(gRPC to external service)"]
        Router["Filter 3: Router"]
        WP --> EP --> Router
    end

    WP -->|gRPC| Authorino["Authorino"]
    WP -->|gRPC| Limitador["Limitador"]
    Authorino -->|HTTP callback| MaaSAPI["MaaS API<br/>/validate"]
    EP -->|gRPC| BBR["BBR Service<br/>port 9004"]
    Router --> Backend["Model Backend"]

    subgraph Kuadrant Operator
        KO["Reconciles AuthPolicy<br/>& TRLP into WasmPlugin"]
    end

    subgraph MaaS Controller
        MC["Reconciles MaaSAuthPolicy<br/>& MaaSSubscription<br/>into Kuadrant CRDs"]
    end

    MC --> KO
```

**Filter chain order:**
```
Request → Kuadrant WasmPlugin (auth + rate limit) → BBR ext_proc (payload processing) → Router → Backend
```

**CRD translation chain:**
```
MaaSAuthPolicy → (MaaS Controller) → AuthPolicy → (Kuadrant Operator) → AuthConfig → Authorino
MaaSSubscription → (MaaS Controller) → TRLP → (Kuadrant Operator) → Limitador config
```

#### Limitations

- WasmPlugin cannot read the request body — model must be in URL path
- Per-model HTTPRoutes required for per-model policy targeting
- Three layers of CRD translation (MaaS → Kuadrant → Authorino/Limitador)
- Kuadrant operator deployment complexity (CSV patching, race conditions)

---

### Approach B: Lightweight ext_proc Before Kuadrant

Adds a third Envoy filter before Kuadrant to extract the model from the body:

```mermaid
graph LR
    Client([Client]) --> Envoy

    subgraph Envoy["Envoy Gateway"]
        direction TB
        LE["Filter 1: Lightweight ext_proc<br/>(body-field-to-header only)"]
        WP["Filter 2: Kuadrant WasmPlugin<br/>(auth + rate limit)"]
        EP["Filter 3: BBR ext_proc<br/>(payload processing)"]
        Router["Filter 4: Router"]
        LE --> WP --> EP --> Router
    end

    LE -->|gRPC| LightBBR["Lightweight BBR<br/>(model extraction)"]
    WP -->|gRPC| Authorino
    WP -->|gRPC| Limitador
    EP -->|gRPC| BBR["BBR Service<br/>(minus body-field-to-header)"]
    Router --> Backend["Model Backend"]
```

**Filter chain order:**
```
Request → Lightweight ext_proc (model→header) → Kuadrant WasmPlugin → BBR ext_proc → Router → Backend
```

#### Pros
- Solves unified entry point — Kuadrant can see model via header
- Minimal change to existing architecture
- Kuadrant policies continue to work

#### Cons
- **Three Envoy filters** — additional latency and complexity
- **Two ext_proc services** to deploy and maintain
- Kuadrant coupling remains — all deployment complexity persists
- Still locked to Authorino + Limitador via Kuadrant
- Does not address the fundamental tight-coupling problem

---

### Approach C: Unified BBR Pipeline (This Proposal)

Consolidate everything into a single BBR ext_proc with pluggable auth and rate-limit plugins:

```mermaid
graph LR
    Client([Client]) --> Envoy

    subgraph Envoy["Envoy Gateway"]
        direction TB
        EP["Single Filter: BBR ext_proc"]
        Router["Router"]
        EP --> Router
    end

    EP -->|gRPC| BBR

    subgraph BBR["BBR Service (single ext_proc)"]
        direction TB
        P1["1. body-field-to-header"]
        P2["2. auth plugin"]
        P3["3. rate-limit plugin"]
        P4["4. model-provider-resolver"]
        P5["5. api-translation"]
        P6["6. apikey-injection"]
        P1 --> P2 --> P3 --> P4 --> P5 --> P6
    end

    P2 -->|gRPC| Authorino["Authorino<br/>(swappable)"]
    Authorino -->|HTTP| MaaSAPI["MaaS API<br/>/validate"]
    P3 -->|gRPC| Limitador["Limitador<br/>(swappable)"]

    Router --> Backend["Model Backend"]

    subgraph MaaS Controller
        MC["Reconciles MaaS CRDs<br/>→ Authorino AuthConfig<br/>(direct, no Kuadrant)"]
    end
```

**Filter chain order:**
```
Request → BBR ext_proc (auth + rate limit + payload processing) → Router → Backend
```

**CRD translation chain (simplified):**
```
MaaSAuthPolicy → (MaaS Controller) → AuthConfig → Authorino    (2 hops, not 3)
MaaSSubscription → (BBR plugin reads directly via informer)     (1 hop)
```

---

## Detailed Design

### Plugin Chain

The BBR ext_proc runs the following plugin chain sequentially. Each plugin implements the standard `RequestProcessor` interface:

```go
type RequestProcessor interface {
    ProcessRequest(ctx context.Context, cycleState *CycleState, 
                   request *InferenceRequest) error
}
```

| Order | Plugin | Type | Responsibility |
|-------|--------|------|---------------|
| 1 | `body-field-to-header` | Built-in (upstream) | Extract `model` from body → `X-Gateway-Model-Name` header |
| 2 | `authorino-auth` | **New** (pluggable) | Call Authorino `Check()` gRPC, write identity to CycleState |
| 3 | `limitador-ratelimit` | **New** (pluggable) | Call Limitador `ShouldRateLimit()` gRPC, enforce token limits |
| 4 | `model-provider-resolver` | Existing | Look up ExternalModel CRD, resolve provider info |
| 5 | `api-translation` | Existing | Translate OpenAI ↔ provider format |
| 6 | `apikey-injection` | Existing | Inject provider-specific auth headers |

### CycleState as the Plugin Contract

Plugins communicate via CycleState keys. This is the interface contract — any auth plugin that writes these keys is compatible with any rate-limit plugin that reads them:

```
Auth plugin WRITES:                Rate-limit plugin READS:
  username                           username
  groups                             model (from body-field-to-header)
  subscription-key                   subscription-key
  subscription-name                  subscription-name
  key-id

Provider-resolver WRITES:          api-translation / apikey-injection READ:
  provider                           provider
  model                              model
  credential-ref-name                credential-ref-name
  credential-ref-namespace           credential-ref-namespace
```

### Auth Plugin: `authorino-auth`

Default implementation that calls Authorino's ext_authz gRPC API:

```go
func (p *AuthorinoAuthPlugin) ProcessRequest(ctx context.Context, 
    cycleState *framework.CycleState, req *framework.InferenceRequest) error {
    
    // 1. Extract Authorization header
    authHeader := req.Headers["authorization"]
    if authHeader == "" {
        return &errcommon.Error{Code: errcommon.Unauthorized, Msg: "missing authorization"}
    }

    // 2. Build ext_authz CheckRequest
    checkReq := &authv3.CheckRequest{
        Attributes: &authv3.AttributeContext{
            Request: &authv3.AttributeContext_Request{
                Http: &authv3.AttributeContext_HttpRequest{
                    Headers: req.Headers,
                    Path:    req.Headers[":path"],
                    Method:  req.Headers[":method"],
                },
            },
        },
    }

    // 3. Call Authorino via gRPC
    resp, err := p.authClient.Check(ctx, checkReq)
    if err != nil {
        return &errcommon.Error{Code: errcommon.Internal, Msg: "auth service unavailable"}
    }

    // 4. Check result
    if resp.Status.Code != int32(codes.OK) {
        return &errcommon.Error{Code: errcommon.Forbidden, Msg: "access denied"}
    }

    // 5. Extract identity from response headers and write to CycleState
    cycleState.Set("username", resp.OkResponse.Headers["x-maas-username"])
    cycleState.Set("groups", resp.OkResponse.Headers["x-maas-group"])
    cycleState.Set("subscription-key", resp.OkResponse.Headers["x-maas-subscription-key"])
    
    return nil // continue to next plugin
}
```

**Error handling:** Returning a non-nil error short-circuits the plugin chain. The BBR framework maps error codes to HTTP status codes (401, 403, 429, etc.) and sends an immediate response to Envoy.

### Rate-Limit Plugin: `limitador-ratelimit`

Default implementation that calls Limitador's RLS gRPC API:

```go
func (p *LimitadorPlugin) ProcessRequest(ctx context.Context,
    cycleState *framework.CycleState, req *framework.InferenceRequest) error {

    // 1. Read identity from CycleState (written by auth plugin)
    username, _ := cycleState.Get("username")
    model, _ := cycleState.Get("model")
    subKey, _ := cycleState.Get("subscription-key")

    // 2. Build RLS request with descriptors
    rlsReq := &rlsv3.RateLimitRequest{
        Domain: "maas",
        Descriptors: []*rlsv3.RateLimitDescriptor{
            {Entries: []*rlsv3.RateLimitDescriptor_Entry{
                {Key: "username", Value: username},
                {Key: "model", Value: model},
                {Key: "subscription_key", Value: subKey},
            }},
        },
    }

    // 3. Call Limitador via gRPC
    resp, err := p.rlsClient.ShouldRateLimit(ctx, rlsReq)
    if err != nil {
        return &errcommon.Error{Code: errcommon.Internal, Msg: "rate limit service unavailable"}
    }

    // 4. Check result
    if resp.OverallCode == rlsv3.RateLimitResponse_OVER_LIMIT {
        return &errcommon.Error{Code: errcommon.TooManyRequests, Msg: "rate limit exceeded"}
    }

    return nil // allowed, continue
}
```

### Pluggability: Swapping Backends

The plugin chain is configured via Helm values. Swapping a backend is a one-line change:

```yaml
# Default: Authorino + Limitador
plugins:
  - type: body-field-to-header
    name: model-extractor
    json:
      fieldName: model
      headerName: X-Gateway-Model-Name
  - type: authorino-auth            # ← default auth plugin
    name: auth
    json:
      endpoint: "authorino-authorino-authorization.auth-system.svc.cluster.local:50051"
  - type: limitador-ratelimit        # ← default rate-limit plugin
    name: rate-limit
    json:
      endpoint: "limitador-limitador.kuadrant-system.svc.cluster.local:8081"
  - type: model-provider-resolver
    name: model-provider-resolver
  - type: api-translation
    name: api-translation
  - type: apikey-injection
    name: apikey-injection
```

```yaml
# Future: Keycloak + OpenMeter
plugins:
  - type: body-field-to-header
    name: model-extractor
    json:
      fieldName: model
      headerName: X-Gateway-Model-Name
  - type: keycloak-auth              # ← swap auth backend
    name: auth
    json:
      endpoint: "keycloak.auth-system.svc.cluster.local:8080"
      realm: "maas"
  - type: openmeter-metering         # ← swap to metering with circuit breaker
    name: metering
    json:
      endpoint: "openmeter.billing.svc.cluster.local:8080"
  - type: model-provider-resolver
    name: model-provider-resolver
  - type: api-translation
    name: api-translation
  - type: apikey-injection
    name: apikey-injection
```

### MaaS Controller Changes

The MaaS controller would stop creating Kuadrant CRDs and instead:

| Current | Proposed |
|---------|----------|
| Creates Kuadrant `AuthPolicy` targeting per-model HTTPRoute | Creates Authorino `AuthConfig` directly (skip Kuadrant middleman) |
| Creates Kuadrant `TokenRateLimitPolicy` targeting per-model HTTPRoute | Configures Limitador limits directly via its API, or BBR rate-limit plugin reads `MaaSSubscription` CRDs via informer |
| Depends on Kuadrant operator to reconcile policies | Direct communication with auth/rate-limit backends |

---

## Sequence Diagrams

### Current Architecture (Approach A) — Model in Path

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Envoy
    participant WP as Kuadrant WasmPlugin<br/>(in-process)
    participant Auth as Authorino
    participant API as MaaS API
    participant Lim as Limitador
    participant BBR as BBR ext_proc
    participant M as Model Backend

    C->>E: POST /llm/granite-3b/v1/chat/completions<br/>Authorization: Bearer sk-oai-xxx<br/>Body: {"model":"granite-3b","messages":[...]}

    Note over E,WP: Filter 1: Kuadrant WasmPlugin

    E->>WP: Evaluate AuthPolicy<br/>(matches path /llm/granite-3b/*)
    WP->>Auth: gRPC Check() — headers + path
    Auth->>API: HTTP POST /internal/v1/api-keys/validate
    API-->>Auth: {valid: true, username: "nitzik", groups: ["team-a"]}
    Auth-->>WP: ALLOW + identity headers

    WP->>Lim: gRPC ShouldRateLimit()<br/>descriptors: {user: nitzik, model: granite-3b}
    Lim-->>WP: OK (under limit)
    WP-->>E: ALLOW — inject X-MaaS-Username, X-MaaS-Group headers

    Note over E,BBR: Filter 2: BBR ext_proc

    E->>BBR: gRPC ProcessRequest — headers + body
    Note over BBR: body-field-to-header → provider-resolver<br/>→ api-translation → apikey-injection
    BBR-->>E: Mutated headers + body

    Note over E,M: Filter 3: Router

    E->>M: Forward to granite-3b-predictor:8080
    M-->>E: Response
    E->>BBR: gRPC ProcessResponse — response body
    Note over BBR: api-translation (reverse)
    BBR-->>E: Translated response
    E-->>C: OpenAI-format response
```

### Proposed Architecture (Approach C) — Model in Body Only

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Envoy
    participant BBR as BBR ext_proc<br/>(single filter)
    participant Auth as Authorino
    participant API as MaaS API
    participant Lim as Limitador
    participant M as Model Backend

    C->>E: POST /v1/chat/completions<br/>Authorization: Bearer sk-oai-xxx<br/>Body: {"model":"granite-3b","messages":[...]}

    Note over E,BBR: Single Filter: BBR ext_proc

    E->>BBR: gRPC ProcessRequest — headers + body

    Note over BBR: Plugin 1: body-field-to-header
    Note over BBR: Extracts "granite-3b" from body<br/>Sets X-Gateway-Model-Name header<br/>Writes "model" to CycleState

    Note over BBR: Plugin 2: authorino-auth
    BBR->>Auth: gRPC Check() — headers (incl. model header)
    Auth->>API: HTTP POST /internal/v1/api-keys/validate
    API-->>Auth: {valid: true, username: "nitzik", groups: ["team-a"]}
    Auth-->>BBR: ALLOW + identity
    Note over BBR: Writes username, groups,<br/>subscription-key to CycleState

    Note over BBR: Plugin 3: limitador-ratelimit
    BBR->>Lim: gRPC ShouldRateLimit()<br/>descriptors from CycleState
    Lim-->>BBR: OK (under limit)

    Note over BBR: Plugin 4-6: provider-resolver → api-translation → apikey-injection
    Note over BBR: (existing plugins, unchanged)

    BBR-->>E: Mutated headers + body + target endpoint

    Note over E,M: Router

    E->>M: Forward to resolved backend
    M-->>E: Response
    E->>BBR: gRPC ProcessResponse
    Note over BBR: api-translation (reverse)
    BBR-->>E: Translated response
    E-->>C: OpenAI-format response
```

### Approach B for Reference — Three Filters

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Envoy
    participant LBR as Lightweight ext_proc
    participant WP as Kuadrant WasmPlugin
    participant Auth as Authorino
    participant Lim as Limitador
    participant BBR as BBR ext_proc
    participant M as Model Backend

    C->>E: POST /v1/chat/completions<br/>Body: {"model":"granite-3b",...}

    Note over E,LBR: Filter 1: Lightweight ext_proc
    E->>LBR: gRPC — headers + body
    Note over LBR: Extracts model → X-Gateway-Model-Name header
    LBR-->>E: Header mutation only

    Note over E,WP: Filter 2: Kuadrant WasmPlugin
    E->>WP: Evaluate policies (can see model header now)
    WP->>Auth: gRPC Check()
    Auth-->>WP: ALLOW
    WP->>Lim: gRPC ShouldRateLimit()
    Lim-->>WP: OK
    WP-->>E: ALLOW

    Note over E,BBR: Filter 3: BBR ext_proc
    E->>BBR: gRPC — headers + body
    Note over BBR: provider-resolver → api-translation → apikey-injection
    BBR-->>E: Mutated headers + body

    E->>M: Forward to backend
    M-->>C: Response (via BBR reverse translation)
```

### Auth Plugin Swap — Future State with Keycloak

```mermaid
sequenceDiagram
    participant C as Client
    participant E as Envoy
    participant BBR as BBR ext_proc
    participant KC as Keycloak
    participant OM as OpenMeter
    participant M as Model Backend

    C->>E: POST /v1/chat/completions<br/>Authorization: Bearer eyJhbGc...<br/>Body: {"model":"granite-3b",...}

    E->>BBR: gRPC ProcessRequest

    Note over BBR: Plugin 1: body-field-to-header

    Note over BBR: Plugin 2: keycloak-auth (swapped)
    BBR->>KC: Token introspection / UserInfo
    KC-->>BBR: {active: true, sub: "nitzik", groups: [...]}
    Note over BBR: Writes identity to CycleState

    Note over BBR: Plugin 3: openmeter-metering (swapped)
    BBR->>OM: Usage check + record
    OM-->>BBR: {allowed: true, remaining: 4500}

    Note over BBR: Plugins 4-6: unchanged

    BBR-->>E: Processed request
    E->>M: Forward
```

---

## Comparison Matrix

| Aspect | A: Current | B: Extra ext_proc | C: Unified BBR (Proposed) |
|--------|-----------|-------------------|--------------------------|
| **Envoy filters** | 2 (WasmPlugin + ext_proc) | 3 (ext_proc + WasmPlugin + ext_proc) | 1 (ext_proc only) |
| **Services to deploy** | Kuadrant operator + Authorino + Limitador + BBR | Same + lightweight BBR | Authorino + Limitador + BBR |
| **Kuadrant dependency** | Required (operator + WasmPlugin) | Required | **Removed** |
| **Model in URL path** | Required | Not required | **Not required** |
| **Body visibility for auth** | No (WasmPlugin can't see body) | No (model promoted to header first) | **Yes** (ext_proc sees body natively) |
| **Per-model HTTPRoutes** | Required for policy targeting | Potentially required | **Not required** |
| **Auth pluggability** | Locked to Authorino via Kuadrant | Locked to Authorino via Kuadrant | **Pluggable** (any auth backend) |
| **Rate-limit pluggability** | Locked to Limitador via Kuadrant | Locked to Limitador via Kuadrant | **Pluggable** (Limitador, metering, etc.) |
| **CRD translation hops** | 3 (MaaS → Kuadrant → Backend) | 3 | **1-2** (MaaS → Backend directly) |
| **Deployment complexity** | High (CSV patching, race conditions) | Higher (+ extra service) | **Low** (standard Helm chart) |
| **Latency (filters)** | 1 WASM eval + 1 gRPC hop | 1 gRPC + 1 WASM + 1 gRPC | **1 gRPC hop** |
| **Backend calls** | Authorino gRPC + Limitador gRPC | Same | Same (from within BBR) |
| **Single point of failure** | BBR down = external models broken | Same | BBR down = everything down |
| **Standards alignment** | WasmPlugin (Istio-specific) | Standard ext_proc | **Standard ext_proc** |
| **Upstream BBR alignment** | Partial (BBR for payload only) | Partial | **Full** (BBR as the vertical) |
| **Change required** | None (status quo) | New lightweight ext_proc + deploy config | New auth + rate-limit plugins + controller update |

---

## Pros and Cons

### Approach C: Unified BBR (This Proposal)

#### Pros

1. **Solves unified entry point natively** — BBR sees the request body, so the model name is available to auth and rate-limit plugins without any workaround. No model in path required.

2. **Removes Kuadrant operator dependency** — Eliminates CSV patching, WasmPlugin management, `MissingDependency` race conditions, and the entire Kuadrant CRD translation layer.

3. **Single Envoy filter** — One gRPC hop from Envoy to BBR, vs. two filters (current) or three filters (Approach B). Lower latency at the filter level.

4. **True pluggability** — Auth and rate-limiting are BBR plugins that can be swapped via Helm values. Future backends (Keycloak, OpenMeter, custom OIDC) are plugin implementations, not architectural changes.

5. **Simpler CRD chain** — MaaS controller talks directly to backends (Authorino AuthConfig, Limitador API) or BBR plugins read MaaS CRDs via informers. Cuts out the Kuadrant middleman.

6. **Aligns with upstream direction** — The 3.5 plan already calls for "Pluggable BBR framework as a vertical." This proposal is the natural conclusion of that direction.

7. **Fewer moving parts** — Drop Kuadrant operator from the deployment. Keep only what's needed: Authorino (auth), Limitador (rate-limit), BBR (orchestration).

8. **Consistent extension model** — Everything in the gateway is a BBR plugin. Auth, rate-limit, payload processing, guardrails (TrustyAI), metering — all follow the same pattern.

#### Cons

1. **Single point of failure** — If BBR goes down, both auth and payload processing are unavailable. Today, Kuadrant WasmPlugin continues to enforce auth even if BBR is down.
   - *Mitigation:* BBR already supports HA replicas. The `failure_mode_allow: false` setting ensures fail-closed behavior. In practice, BBR being down already breaks external model support today.

2. **Non-standard auth pattern** — Using ext_proc for authentication is not the canonical Envoy pattern (that would be ext_authz). Security auditors may question this.
   - *Mitigation:* ext_proc is a standard Envoy API. The BBR auth plugin still calls Authorino's standard ext_authz gRPC API — the call just originates from BBR instead of from Envoy's filter chain.

3. **New plugin development** — Two new BBR plugins need to be built (auth + rate-limit).
   - *Mitigation:* These are straightforward gRPC client calls with CycleState integration. The provider-resolver and apikey-injection plugins already demonstrate the pattern. Estimated effort: ~1-2 weeks per plugin.

4. **MaaS controller refactor** — Controller needs to stop creating Kuadrant CRDs and start creating Authorino AuthConfig or configuring Limitador directly.
   - *Mitigation:* This actually simplifies the controller — fewer CRDs to manage, fewer translation layers.

5. **Loss of Kuadrant policy model** — Kuadrant provides sophisticated policy merging (gateway-level defaults overridden by route-level policies). This logic would need to be replicated in BBR plugins or the MaaS controller.
   - *Mitigation:* The current MaaS controller already handles policy aggregation (many-to-many MaaSAuthPolicy → single AuthPolicy per model). The Kuadrant layer adds little beyond translation.

6. **Upstream framework changes may be needed** — The BBR plugin interface may need minor extensions (e.g., formal `AuthPlugin` interface, richer error response bodies).
   - *Mitigation:* Current `RequestProcessor` interface already supports returning typed errors that map to HTTP status codes. Minimal changes expected.

---

## Migration Path

### Phase 1: Build Plugins (No Production Impact)
- Implement `authorino-auth` plugin in `ai-gateway-payload-processing`
- Implement `limitador-ratelimit` plugin in `ai-gateway-payload-processing`
- Register plugins in `cmd/main.go`
- Unit and integration tests

### Phase 2: Validate in Dev Environment
- Deploy BBR with full plugin chain (auth + rate-limit + payload processing)
- Remove Kuadrant WasmPlugin from the gateway
- Validate unified entry point (`/v1/chat/completions` with model in body)
- Performance testing: measure latency difference

### Phase 3: Update MaaS Controller
- Stop creating Kuadrant AuthPolicy and TokenRateLimitPolicy CRDs
- Create Authorino AuthConfig directly, or have BBR plugins read MaaS CRDs via informers
- Update deployment scripts to remove Kuadrant operator installation

### Phase 4: Production Rollout
- Deploy alongside existing architecture (feature flag / separate gateway)
- Gradual migration of models to new flow
- Deprecate Kuadrant-based flow

---

## Open Questions

1. **Routing without per-model HTTPRoutes** — If there's no model in the path, how does the Envoy router know which backend to forward to? Options:
   - BBR sets a target endpoint header and a single HTTPRoute uses header-based matching
   - BBR uses ext_proc's dynamic routing capability to set the upstream cluster
   - A single catch-all HTTPRoute forwards all `/v1/*` to a load balancer that BBR has configured

2. **Subscription selection** — Today, the AuthPolicy calls `maas-api /internal/v1/subscriptions/select` as a metadata step. In the new model, should this be a separate plugin or part of the auth plugin?

3. **Telemetry** — Today, Kuadrant's TelemetryPolicy extracts metrics labels from `auth.identity`. In the new model, how do we expose per-request metrics (model, user, subscription, tier)?

4. **Gateway-level default deny** — Today, a gateway-level TokenRateLimitPolicy with 0-token rate blocks all unsubscribed models. In the new model, the rate-limit plugin would need equivalent default-deny logic.

5. **Upstream BBR framework changes** — Do we need to propose changes to the `gateway-api-inference-extension` BBR framework to better support auth/rate-limit plugin patterns?

6. **Multiple gateways** — If multiple gateways exist (e.g., internal + external), each with different auth requirements, how do per-gateway plugin chains work?

---

## Components Changed

| Component | Changes |
|-----------|---------|
| `ai-gateway-payload-processing` | New `authorino-auth` and `limitador-ratelimit` plugins, updated `cmd/main.go`, updated Helm chart |
| `models-as-a-service/maas-controller` | Stop creating Kuadrant CRDs. Create Authorino AuthConfig directly or configure BBR plugins via CRD/ConfigMap |
| `models-as-a-service/deployment` | Remove Kuadrant operator installation, WasmPlugin, CSV patching. Simplify deploy script |
| `gateway-api-inference-extension` | Potential upstream proposals for auth/rate-limit plugin patterns in BBR framework |

---

## References

- RHAIRFE-1304: Unified entry point to Inference Gateway
- RHAIRFE-1898: Intelligent model selection via BBR
- RHAIRFE-1901: TrustyAI Request Guard via BBR
- [Inference Gateway Plan - 3.5 and beyond](/Users/nitzikow/Downloads/Inference%20Gateway%20Plan%20-%203.5%20and%20beyond.pdf)
- [BBR Plugin Framework Proposal (upstream)](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/1964-pluggable-bbr-framework)
- [Envoy ext_proc protocol](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/ext_proc/v3/ext_proc.proto)
