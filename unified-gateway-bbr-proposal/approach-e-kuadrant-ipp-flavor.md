# Approach E: Kuadrant IPP Flavor

**Date:** May 1, 2026
**Author:** Noy Itzikowitz
**Context:** Follow-up to [Proposal: Unified Gateway Flow via Pluggable IPP](https://docs.google.com/document/d/124eRK29AApcjPSQBuOtbEbJfPr13T9kzlBasqXozSGE/edit?usp=sharing), based on AI Gateway F2F conclusions and cross-team feedback.

---

## Summary

Following the AI Gateway F2F in Boston and discussions with the RHCL team, this document proposes a two-phase approach to the unified entry point:

- **Phase 1 (RHOAI 3.5, short-term):** Approach C from the original proposal — enable body inspection in Kuadrant's WasmPlugin so the model name can be read from the request body. Minimal change, unblocks the unified entry point.

- **Phase 2 (RHOAI 3.6+, long-term): Approach E** — a new Kuadrant deployment flavor where auth and rate limiting are deployed as IPP plugins, **built and maintained by the Kuadrant team**. The Kuadrant control plane (operator, CRDs, lifecycle management) remains unchanged. Authorino and Limitador internal APIs remain closed.

This document focuses on Approach E — its architecture, feasibility, and what it takes to get there.

---

## Why Approach E?

Approach C (body inspection in WasmPlugin) solves the immediate unified entry point for 3.5, but leaves two issues:

1. **Body parsed twice on every request** — the WasmPlugin buffers and parses the request body to extract the model name, then IPP (ext_proc) parses the same body again for payload processing (api-translation, provider-resolver, etc.)

2. **Body parsed twice on every response** — today, the WasmPlugin opens the response body to extract token usage counts for Limitador reporting, and IPP also opens the response body for api-translation (translating provider response format back to OpenAI). Two separate components parsing the same response.

Approach E consolidates all body processing — auth, rate limiting, and payload processing — into a single IPP ext_proc, eliminating the double parsing on both request and response paths.

---

## How It Works Today

```
┌────────────────────────────────┐     ┌────────────────────────────┐
│ Kuadrant WasmPlugin            │     │ IPP ext_proc               │
│ (Rust WASM, in-process)        │     │ (Go/Rust, external gRPC)   │
│                                │     │                            │
│ 1. Route matching (CEL)        │     │ 1. body-field-to-header    │
│ 2. gRPC → Authorino Check()   │────►│ 2. provider-resolver       │
│ 3. gRPC → Limitador Check()   │     │ 3. api-translation         │
│ 4. Parse response body         │     │ 4. apikey-injection        │
│    → extract token counts      │     │                            │
│ 5. gRPC → Limitador Report()  │     │                            │
│                                │     │                            │
│ Config: JSON blob from         │     │ Config: Helm values        │
│ Kuadrant operator via          │     │ (plugin chain)             │
│ WasmPlugin CRD                 │     │                            │
└────────────────────────────────┘     └────────────────────────────┘
         ▲                                      ▲
         │                                      │
    Kuadrant Operator                    IPP Helm chart
    (reconciles AuthPolicy,
     TRLP → WasmPlugin config
     + AuthConfig + Limitador limits)
```

### The Configuration Flow (Current)

1. MaaS admin creates `MaaSAuthPolicy` and `MaaSSubscription`
2. MaaS controller translates them into Kuadrant `AuthPolicy` and `TokenRateLimitPolicy`
3. Kuadrant operator reconciles these into:
   - `AuthConfig` CRD (for Authorino)
   - Limitador limits YAML (via ConfigMap)
   - WasmPlugin CRD containing a JSON configuration blob with:
     - Service endpoints (Authorino, Limitador cluster DNS)
     - Action sets (route conditions, CEL predicates, data bindings)
     - Service types and failure modes
4. Istio loads the WasmPlugin and passes the JSON config to the wasm-shim at startup
5. At request time, wasm-shim evaluates route conditions, makes gRPC calls to Authorino/Limitador, and enforces policies

### What the WasmPlugin Config Looks Like

The Kuadrant operator generates a JSON blob that defines the complete policy evaluation pipeline:

```json
{
  "services": {
    "auth-service": {
      "endpoint": "kuadrant-auth-service",
      "type": "auth",
      "failureMode": "deny",
      "timeout": "200ms"
    },
    "ratelimit-check-service": {
      "endpoint": "kuadrant-ratelimit-service",
      "type": "ratelimit-check",
      "failureMode": "allow"
    },
    "ratelimit-report-service": {
      "endpoint": "kuadrant-ratelimit-service",
      "type": "ratelimit-report",
      "failureMode": "allow"
    }
  },
  "actionSets": [
    {
      "routeRuleConditions": {
        "hostnames": ["maas.example.com"],
        "predicates": ["request.method == 'POST'"]
      },
      "actions": [
        {
          "service": "auth-service",
          "scope": "auth-config-for-granite-3b"
        },
        {
          "service": "ratelimit-check-service",
          "scope": "maas-namespace",
          "conditionalData": [
            {
              "data": [
                {"key": "limit-id", "value": "1"},
                {"key": "ratelimit.hits_addend", "value": "0"}
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

---

## Approach E: The Kuadrant IPP Flavor

Instead of generating a WasmPlugin CRD, the Kuadrant operator generates **IPP plugin configuration** — and the auth/rate-limit logic runs as IPP plugins inside the same ext_proc process as payload processing.

```
┌──────────────────────────────────────────────────────┐
│ IPP ext_proc (single process)                        │
│                                                      │
│ 1. body-field-to-header                              │
│ 2. kuadrant-auth plugin ← built by Kuadrant team     │
│    → gRPC to Authorino Check()                       │
│ 3. kuadrant-ratelimit plugin ← built by Kuadrant team│
│    → gRPC to Limitador Check()                       │
│ 4. model-provider-resolver                           │
│ 5. api-translation                                   │
│ 6. apikey-injection                                  │
│ 7. kuadrant-usage-report plugin (response phase)     │
│    → extracts token counts from response body        │
│    → gRPC to Limitador Report()                      │
│                                                      │
│ Config: Kuadrant operator pushes plugin config       │
│ via ConfigMap (same action set format)               │
└──────────────────────────────────────────────────────┘
         ▲                        ▲
         │                        │
    Kuadrant Operator         Authorino + Limitador
    (reconciles same CRDs,    (unchanged, same pods,
     generates IPP config     same gRPC APIs)
     instead of WasmPlugin)
```

### What Changes for the Kuadrant Operator

The operator already generates the complete policy configuration as a JSON blob for the wasm-shim. In Approach E, it would generate a **similar configuration** but targeting IPP plugin format instead of WasmPlugin format.

| Current (WasmPlugin) | Approach E (IPP) |
|----------------------|------------------|
| Generates `WasmPlugin` CRD with inline JSON config | Generates `ConfigMap` with IPP plugin config |
| Config consumed by wasm-shim via `on_configure()` | Config consumed by IPP plugins via mounted volume or gRPC discovery |
| Route matching via CEL in wasm-shim | Route matching via CEL in IPP auth plugin |
| Auth scope maps to AuthConfig name | Auth scope maps to AuthConfig name (same) |
| Rate limit descriptors built from CEL expressions | Rate limit descriptors built from CEL expressions (same) |
| Token usage extracted from response body by wasm-shim | Token usage extracted from response body by IPP usage-report plugin |

### What the Kuadrant Team Builds

Three IPP plugins, all owned and maintained by the Kuadrant team:

**1. `kuadrant-auth` plugin** (implements `RequestProcessor`)
- Receives action set configuration from Kuadrant operator (via ConfigMap)
- Evaluates route conditions and CEL predicates (same logic as wasm-shim)
- Makes gRPC `Check()` call to Authorino (same ext_authz protocol)
- Writes auth result (identity, groups, subscription) to CycleState
- On failure: short-circuits with 401/403

**2. `kuadrant-ratelimit` plugin** (implements `RequestProcessor`)
- Reads auth identity from CycleState
- Evaluates rate-limit predicates and builds descriptors (same CEL expressions)
- Makes gRPC `CheckRateLimit()` call to Limitador (Kuadrant RLS protocol)
- On OVER_LIMIT: short-circuits with 429

**3. `kuadrant-usage-report` plugin** (implements `ResponseProcessor`)
- Extracts token usage from response body (same JSON pointer extraction as wasm-shim's `TokenUsageTask`)
- Makes gRPC `Report()` call to Limitador with actual token count
- This is where the body-parsed-twice problem gets solved — the response body is already available in IPP, no need for wasm-shim to open it separately

### What Stays the Same

- **Authorino** — same pod, same gRPC API, same AuthConfig CRDs. No changes.
- **Limitador** — same pod, same gRPC API, same limits configuration. No changes.
- **Kuadrant Operator control plane** — same CRDs (AuthPolicy, TRLP), same reconciliation logic. Only the output target changes (ConfigMap instead of WasmPlugin).
- **MaaS CRDs** — MaaSAuthPolicy, MaaSSubscription unchanged. MaaS controller still creates Kuadrant policies.
- **APIs remain closed** — the IPP plugins call the same gRPC endpoints that the wasm-shim calls. No new public API surface.

---

## Feasibility Assessment

### Is This Technically Doable?

**Yes.** Based on analysis of all repos involved:

**1. The gRPC protocols are standard and documented:**
- Authorino implements `envoy.service.auth.v3.Authorization` (standard Envoy ext_authz)
- Limitador implements both standard Envoy RLS and Kuadrant's custom `kuadrant.service.ratelimit.v1.RateLimitService`
- Proto definitions are available in the Limitador repo (`limitador-server/proto/kuadrantrls.proto`)
- An IPP plugin can make the same gRPC calls using the same protobuf message formats

**2. The IPP plugin framework supports this pattern:**
- Plugins are initialized at startup — can create long-lived gRPC clients
- `CycleState` enables inter-plugin data passing (auth result → rate-limit → payload processing)
- Plugins can return typed errors that map to HTTP status codes (401, 403, 429)
- Both `RequestProcessor` and `ResponseProcessor` interfaces are available

**3. The configuration format is already structured:**
- The Kuadrant operator already generates a well-defined JSON configuration (action sets, services, predicates)
- This same format can be written to a ConfigMap instead of a WasmPlugin CRD
- IPP plugins can load this configuration from a mounted ConfigMap volume
- Hot-reload via ConfigMap watch is a standard Kubernetes pattern

**4. CEL evaluation is available in both Go and Rust:**
- The wasm-shim evaluates CEL expressions for route matching and descriptor building
- The same CEL expressions can be evaluated in IPP plugins
- Go has `google/cel-go`, Rust has `clarkmcc/cel-interpreter`

### What Are the Risks?

**1. Configuration format compatibility**
The IPP plugins need to consume the same JSON format the operator generates. If the operator's output format changes, both wasm-shim and IPP plugins need to be updated. Mitigation: the Kuadrant team owns both, so they control the contract.

**2. Feature parity with wasm-shim**
The wasm-shim has significant logic: route matching, CEL evaluation, async gRPC handling, response body streaming (SSE parser for token extraction), descriptor building. The IPP plugins need to replicate this. Mitigation: the IPP plugins can start with the subset needed for MaaS and expand over time.

**3. Two deployment models to support**
Kuadrant would need to support both the wasm-shim model (for non-AI-Gateway customers) and the IPP model (for AI Gateway / MaaS). Mitigation: the control plane and backends are shared, only the data plane runtime differs.

---

## Comparison: Approach C vs Approach E

| Aspect | C: WasmPlugin Body Inspection (3.5) | E: Kuadrant IPP Flavor (3.6+) |
|--------|-------------------------------------|-------------------------------|
| **Envoy filters** | 2 (WasmPlugin + ext_proc) | 1 (ext_proc only) |
| **Request body parsing** | Twice (WasmPlugin + IPP) | Once (IPP only) |
| **Response body parsing** | Twice (WasmPlugin for tokens + IPP for api-translation) | Once (IPP handles both) |
| **Kuadrant control plane** | Yes | Yes |
| **Kuadrant APIs closed** | Yes | Yes |
| **Auth/RL pluggability** | Locked via Kuadrant | Pluggable (IPP plugin interface) |
| **Kuadrant team effort** | Enable body inspection in wasm-shim | Build 3 IPP plugins |
| **MaaS changes** | Minimal | Minimal |
| **Timeline** | Weeks | Months |
| **Recommended for** | **RHOAI 3.5** | **RHOAI 3.6+** |

---

## What Each Team Does

### Phase 1 (RHOAI 3.5): Approach C

| Team | Work |
|------|------|
| **Kuadrant/RHCL** | Enable body inspection in wasm-shim for model name extraction |
| **MaaS** | Update HTTPRoutes and controllers to remove model from URL path |
| **AI Gateway / IPP** | No changes to IPP |

### Phase 2 (RHOAI 3.6+): Approach E

| Team | Work |
|------|------|
| **Kuadrant/RHCL** | Build `kuadrant-auth`, `kuadrant-ratelimit`, and `kuadrant-usage-report` IPP plugins. Update operator to generate ConfigMap-based IPP config as an alternative to WasmPlugin. |
| **MaaS** | No additional changes (already on unified entry point from Phase 1) |
| **AI Gateway / IPP** | Support for Kuadrant plugins in IPP Helm chart. Plugin configuration loading from ConfigMap. |

---

## Open Questions

1. **Plugin language** — The IPP framework has implementations in both Go and Rust. Which should the Kuadrant IPP plugins use? The wasm-shim is Rust, so Rust might be natural for the Kuadrant team. The current MaaS IPP plugins (api-translation, provider-resolver) are in Go.

2. **Configuration delivery** — ConfigMap with mounted volume, or a gRPC discovery service that the IPP plugins call back to the operator? The operator already has a gRPC descriptor service (`kuadrant.v1.DescriptorService`) that the wasm-shim calls for dynamic service discovery.

3. **Scope for Phase 2** — Should the IPP plugins replicate the full wasm-shim feature set (all CEL evaluation, all policy types), or start with the MaaS-specific subset (AuthPolicy + TokenRateLimitPolicy only)?

4. **Shared binary or separate** — Should the Kuadrant IPP plugins be compiled into the same binary as the MaaS IPP plugins, or deployed as a separate IPP instance with its own plugin chain?

5. **MCP Gateway alignment** — The Kuadrant team is also building an MCP Gateway using ext_proc (because WASM has shortcomings around stopping iteration). Could the Kuadrant IPP plugins serve both MaaS and MCP Gateway use cases?
