# vSR HTTP Plugin for BBR — Implementation Plan

**Author**: Noy Itzikowitz
**Date**: 2026-03-10
**Workstream**: 5 (Implement vSR plugin and run e2e tests)
**Reference**: [Pluggable BBR (IGW) + vSR integration doc](https://docs.google.com/document/d/1_pluggable-bbr-vsr-integration)

---

## Context

This plan covers the implementation of a BBR `RequestProcessor` plugin that calls vSR's `POST /v1/route` HTTP endpoint and mutates request headers/body with the routing decision. This enables the IGW Body-Based Router (BBR) to delegate model selection to the vLLM Semantic Router (vSR).

### Current State

| Component | Status |
|-----------|--------|
| Upstream GIE pluggable BBR framework (PRs #2369, #2442) | **Merged** into GIE main (March 3-5, 2026) |
| vSR HTTP API — `POST /v1/route` (Workstream 2) | **Implemented** on branch [`feature/http-routing-api`](https://github.com/noyitz/semantic-router/tree/feature/http-routing-api) & tested locally |
| vSR plugin for BBR (this plan) | **Not started** |
| API-key injection plugin (Workstream 4) | Separate follow-up |

### Architecture

```
                              Envoy Gateway (with Istio)
                             /            |              \
                            /             |               \
                     1. ext-proc     2. ext-proc      3. forward
                        call            call            request
                       /                  |                \
                      v                   v                 v
                    BBR                  EPP           Model Server
               (model selection)   (endpoint picking)
                    |
                    |-- vSR plugin (RequestProcessor)
                    |     calls vSR /v1/route over HTTP
                    |
                    +-- [future] api-key injection plugin
```

The Gateway orchestrates calls in a **star topology** (not a chain):
1. **Gateway -> BBR**: Calls BBR via ext-proc. BBR runs request plugins (e.g., vSR) to determine the target model, sets `X-Gateway-Model-Name` header. Response returns to Gateway.
2. **Gateway -> EPP**: Calls EPP via ext-proc. EPP uses the model header to select the optimal backend pod (KV cache, load, P/D). Response returns to Gateway.
3. **Gateway -> Model Server**: Forwards the request to the selected endpoint.

Response flows the reverse path: Model Server -> Gateway -> EPP (response plugins) -> Gateway -> BBR (response plugins) -> Gateway -> Client.

**Deployment Model**: BBR and vSR run as two containers on the same pod. The vSR plugin calls vSR over localhost HTTP (`http://127.0.0.1:8080`), avoiding network hops.

---

## Prerequisites

### 1. Update GIE Dependency in llm-d-inference-scheduler

The current `go.mod` pins GIE at `v0.0.0-20260128235548-fd30cb97714a` (Jan 28, 2026), which predates the pluggable BBR merge. Update to a post-March 5 commit to get `pkg/bbr/framework`:

```bash
cd llm-d-inference-scheduler
git checkout -b feature/bbr-vsr-plugin
go get sigs.k8s.io/gateway-api-inference-extension@main
go mod tidy
```

### 2. Upstream BBR Plugin Framework Overview

The plugin must implement interfaces from `sigs.k8s.io/gateway-api-inference-extension/pkg/bbr/framework`:

| Type | Description |
|------|-------------|
| `BBRPlugin` | Alias for `plugin.Plugin` — provides `TypedName()` |
| `RequestProcessor` | Embeds `BBRPlugin` + `ProcessRequest(ctx, *InferenceRequest) error` |
| `InferenceRequest` | Has `Headers map[string]string`, `Body map[string]any`, plus mutation methods: `SetHeader()`, `RemoveHeader()` |
| `FactoryFunc` | `func(name string, params json.RawMessage) (BBRPlugin, error)` |
| Registry | `framework.Register(pluginType, factory)` — global plugin factory map |

**Plugin execution**: BBR Server calls `runRequestPlugins()` which iterates request plugins sequentially, calling `ProcessRequest(ctx, req)` on each. Plugins mutate the shared `InferenceRequest` in-place.

**Example upstream plugin**: `pkg/bbr/plugins/body_field_to_header.go` — reads a body field and sets it as a header via `req.SetHeader()`.

---

## Directory Structure

New code lives under `pkg/bbr/` in `llm-d-inference-scheduler`, separate from EPP plugins in `pkg/plugins/`:

```
llm-d-inference-scheduler/
  pkg/
    bbr/                           # NEW - BBR plugins
      vsr/
        types.go                   # vSR request/response Go structs
        vsr_client.go              # HTTP client for vSR /v1/route
        vsr_client_test.go         # Client unit tests
        vsr_plugin.go              # Plugin struct, ProcessRequest, factory
        vsr_plugin_test.go         # Plugin unit tests
      register.go                  # RegisterBBRPlugins()
    plugins/                       # Existing EPP plugins (unchanged)
      ...
```

---

## Implementation Details

### Step 1: vSR Request/Response Types (`pkg/bbr/vsr/types.go`)

```go
// VSRRouteRequest - sent to POST /v1/route
type VSRRouteRequest struct {
    Model    string               `json:"model"`
    Messages []map[string]any     `json:"messages"`
    Metadata *VSRRequestMetadata  `json:"metadata,omitempty"`
}

type VSRRequestMetadata struct {
    Headers map[string]string `json:"headers,omitempty"`
}

// VSRRouteResponse - received from vSR
type VSRRouteResponse struct {
    RoutingDecision  RoutingDecision `json:"routing_decision"`
    Security         *SecurityInfo   `json:"security,omitempty"`
    RequestID        string          `json:"request_id,omitempty"`
    ProcessingTimeMs int64           `json:"processing_time_ms,omitempty"`
}

type RoutingDecision struct {
    SelectedModel        string   `json:"selected_model"`
    DecisionName         string   `json:"decision_name,omitempty"`
    DecisionConfidence   float64  `json:"decision_confidence,omitempty"`
    SelectedCategory     string   `json:"selected_category,omitempty"`
    ReasoningMode        string   `json:"reasoning_mode,omitempty"`
    SelectedModality     []string `json:"selected_modality,omitempty"`
    SelectionMethod      string   `json:"selection_method,omitempty"`
    InjectedSystemPrompt bool     `json:"injected_system_prompt,omitempty"`
}

type SecurityInfo struct {
    JailbreakDetected bool     `json:"jailbreak_detected"`
    PIIDetected       bool     `json:"pii_detected"`
    PIIEntities       []string `json:"pii_entities,omitempty"`
}
```

### Step 2: HTTP Client (`pkg/bbr/vsr/vsr_client.go`)

```go
type VSRClient struct {
    httpClient *http.Client
    baseURL    string
}

func NewVSRClient(baseURL string, timeout time.Duration) *VSRClient
func (c *VSRClient) Route(ctx context.Context, req *VSRRouteRequest) (*VSRRouteResponse, error)
```

**HTTP client configuration** (optimized for localhost):
- Timeout: 5s default (configurable) — vSR typically responds in 10-50ms
- Connection pooling: `MaxIdleConns: 10`, `MaxIdleConnsPerHost: 10`, `IdleConnTimeout: 90s`
- No TLS (plain HTTP over localhost)
- Keep-alives enabled for connection reuse
- No proxy (`Transport.Proxy: nil`)
- Uses `http.NewRequestWithContext` for cancellation support

### Step 3: Plugin Implementation (`pkg/bbr/vsr/vsr_plugin.go`)

```go
const VSRPluginType = "vsr-http-router"

type vsrPluginConfig struct {
    BaseURL string `json:"baseURL"` // default: "http://127.0.0.1:8080"
    Timeout string `json:"timeout"` // default: "5s"
}

type VSRPlugin struct {
    typedName plugin.TypedName
    client    *VSRClient
}

// Compile-time interface assertion
var _ framework.RequestProcessor = &VSRPlugin{}
```

**Factory function**:
```go
func VSRPluginFactory(name string, rawParams json.RawMessage) (framework.BBRPlugin, error) {
    cfg := vsrPluginConfig{BaseURL: "http://127.0.0.1:8080", Timeout: "5s"}
    if rawParams != nil {
        if err := json.Unmarshal(rawParams, &cfg); err != nil {
            return nil, fmt.Errorf("failed to parse vSR plugin config: %w", err)
        }
    }
    timeout, err := time.ParseDuration(cfg.Timeout)
    if err != nil {
        return nil, fmt.Errorf("invalid timeout %q: %w", cfg.Timeout, err)
    }
    return &VSRPlugin{
        typedName: plugin.TypedName{Type: VSRPluginType, Name: name},
        client:    NewVSRClient(cfg.BaseURL, timeout),
    }, nil
}
```

**Key assumption**: Model names are globally unique across in-cluster and external models. If a request specifies a specific model name (not `"auto"`), that name unambiguously identifies the target — no vSR routing is needed.

**ProcessRequest** — core logic:
```go
const AutoModelName = "auto"

func (p *VSRPlugin) ProcessRequest(ctx context.Context, req *framework.InferenceRequest) error {
    // 0. Skip vSR if model is already specified (not "auto")
    //    Model names are unique — a specific model name means the caller
    //    already knows the target. Only "auto" requires vSR routing.
    modelVal, ok := req.Body["model"]
    if ok {
        if modelStr, isStr := modelVal.(string); isStr && modelStr != AutoModelName {
            // Model explicitly set — pass through without calling vSR
            req.SetHeader("X-Gateway-Model-Name", modelStr)
            return nil
        }
    }

    // 1. Build vSR request from req.Body and req.Headers
    vsrReq := buildVSRRequest(req.Body, req.Headers)

    // 2. Call vSR /v1/route
    vsrResp, err := p.client.Route(ctx, vsrReq)
    if err != nil {
        return fmt.Errorf("vSR routing failed: %w", err)
    }

    // 3. Mutate model in body
    req.Body["model"] = vsrResp.RoutingDecision.SelectedModel

    // 4. Set X-Gateway-Model-Name (used by Envoy for route matching)
    req.SetHeader("X-Gateway-Model-Name", vsrResp.RoutingDecision.SelectedModel)

    // 5. Set vSR metadata headers for observability
    req.SetHeader("X-VSR-Decision-Name", vsrResp.RoutingDecision.DecisionName)
    if vsrResp.RoutingDecision.ReasoningMode != "" {
        req.SetHeader("X-VSR-Reasoning-Mode", vsrResp.RoutingDecision.ReasoningMode)
    }
    if vsrResp.RoutingDecision.SelectedCategory != "" {
        req.SetHeader("X-VSR-Selected-Category", vsrResp.RoutingDecision.SelectedCategory)
    }

    // 6. Security headers
    if vsrResp.Security != nil && vsrResp.Security.JailbreakDetected {
        req.SetHeader("X-VSR-Jailbreak-Detected", "true")
    }
    if vsrResp.Security != nil && vsrResp.Security.PIIDetected {
        req.SetHeader("X-VSR-PII-Detected", "true")
    }

    return nil
}
```

**buildVSRRequest helper**:
- Extract `model` from `body["model"]` (string type assertion)
- Extract `messages` from `body["messages"]` ([]any -> []map[string]any)
- Pass incoming `headers` map as `metadata.headers` (enables vSR to use auth headers like `X-Authz-User-Id` for decision-making)

### Step 4: BBR Plugin Registration (`pkg/bbr/register.go`)

```go
package bbr

import (
    "sigs.k8s.io/gateway-api-inference-extension/pkg/bbr/framework"
    "github.com/llm-d/llm-d-inference-scheduler/pkg/bbr/vsr"
)

func RegisterBBRPlugins() {
    framework.Register(vsr.VSRPluginType, vsr.VSRPluginFactory)
}
```

---

## Header Mutation Summary

| Header | Value | Purpose |
|--------|-------|---------|
| `X-Gateway-Model-Name` | `SelectedModel` | Envoy route matching (standard BBR header) |
| `X-VSR-Decision-Name` | `DecisionName` | Routing decision name for observability |
| `X-VSR-Reasoning-Mode` | `"on"` / `"off"` | Downstream reasoning mode (optional) |
| `X-VSR-Selected-Category` | Category name | Selected domain category (optional) |
| `X-VSR-Jailbreak-Detected` | `"true"` | Security flag (only set when detected) |
| `X-VSR-PII-Detected` | `"true"` | Security flag (only set when detected) |

**Body mutation**: `body["model"]` updated from `"auto"` to `SelectedModel` from vSR.

> **Note**: Body mutation support in the upstream pluggable BBR is not yet merged into GIE main (PR in progress by @nirrozenbaum). Header mutations work today. Body mutation will be available shortly — until then, the plugin can be tested with header mutations only.

---

## Unit Test Plan

### `vsr_client_test.go` (using `httptest.NewServer`)

| Test | Description |
|------|-------------|
| `TestRoute_Success` | Mock vSR returns valid response, verify parsed correctly |
| `TestRoute_HTTPError` | Mock returns 4xx/5xx, verify error wrapping |
| `TestRoute_InvalidJSON` | Mock returns non-JSON, verify parse error |
| `TestRoute_Timeout` | Context cancelled, verify deadline exceeded |

### `vsr_plugin_test.go` (using `httptest.NewServer` + `framework.NewInferenceRequest`)

| Test | Description |
|------|-------------|
| `TestProcessRequest_SkipWhenModelSpecified` | `model: "gpt-4"` (not "auto") — verify vSR is NOT called, header set to "gpt-4" |
| `TestProcessRequest_SuccessfulRouting` | `model: "auto"` — verify vSR called, `body["model"]` updated and `X-Gateway-Model-Name` set |
| `TestProcessRequest_HeadersPassthrough` | Verify incoming headers sent as `metadata.headers` to vSR |
| `TestProcessRequest_SecurityHeaders` | vSR returns jailbreak=true, verify `X-VSR-Jailbreak-Detected` set |
| `TestProcessRequest_OptionalFieldsEmpty` | No reasoning_mode, verify no `X-VSR-Reasoning-Mode` header |
| `TestProcessRequest_VSRUnreachable` | vSR down, verify error returned |
| `TestProcessRequest_MissingModel` | No model in body, verify graceful handling |
| `TestProcessRequest_MutationTracking` | Verify `req.MutatedHeaders()` contains expected keys |

Run tests: `cd llm-d-inference-scheduler && go test ./pkg/bbr/...`

---

## E2E Test Plan

### Phase 1: Local Validation (no cluster)
- Unit tests with `httptest.NewServer` mock
- No cluster needed, runs on dev machine

### Phase 2: RHOAI Cluster Validation

**Cluster**: RHOAI on OCP on AWS q7h9j (expires 2026-03-30)
- vSR already deployed in `vllm-semantic-router-system` namespace (simulator mode)

**Flow**: `Client -> Istio Gateway -> BBR (vSR plugin) -> EPP -> Model Server`

**Steps**:
1. Build BBR container image with vSR plugin registered
2. Deploy BBR pod with vSR sidecar container on the cluster
3. Configure Istio Gateway + HTTPRoute to route to BBR
4. Send test requests and verify the full flow

**Test Scenarios**:

| # | Scenario | Expected Result |
|---|----------|-----------------|
| 1 | `model: "auto"` chat completion | vSR selects model, body.model updated, `X-Gateway-Model-Name` set |
| 6 | `model: "gpt-4"` (specific model) | vSR skipped, `X-Gateway-Model-Name` set to "gpt-4" directly |
| 2 | Math question | vSR routes to math-specialized model |
| 3 | Request with auth headers | `X-Authz-User-Id` forwarded to vSR via metadata.headers |
| 4 | vSR sidecar down | BBR returns error (503) |
| 5 | High concurrency (100 rps) | Latency under 100ms p99 for BBR+vSR hop |

---

## Integration with BBR Binary

The upstream GIE `cmd/bbr/runner/runner.go` has `registerInTreePlugins()` that registers built-in plugins. To use the vSR plugin, we need one of:

1. **Extend the BBR binary in llm-d**: Create `cmd/bbr/main.go` that imports and calls `bbr.RegisterBBRPlugins()` before starting the runner. This registers the vSR factory so it can be specified via CLI flags.
2. **Use CLI flag**: The runner supports `--requestPlugins` flag to specify plugin types at startup. Once the vSR factory is registered, specify `--requestPlugins=vsr-http-router`.

This is a follow-up integration step after the plugin code itself is written and tested.

---

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| HTTP latency over localhost | vSR typically 10-50ms. Connection pooling + keep-alives minimize overhead. Plan B: in-process vSR calls (separate effort) |
| Body parsing type assertions | Careful handling of `map[string]any` types from generic JSON unmarshal. Tests cover edge cases. |
| BBR body mutation not yet in GIE main | Nir is pushing the body mutation PR shortly. Header mutations work today — start testing with headers, add body mutation once merged. |
| GIE BBR framework API changes | PR #2450 (response plugins) is still draft but doesn't affect RequestProcessor. Monitor for breaking changes. |
| PR #2413 (import verification) still open | May add constraints on BBR<->EPP imports. Keep `pkg/bbr/` cleanly separated. |

---

## Branch Strategy

- **Repo**: `llm-d-inference-scheduler`
- **Branch**: `feature/bbr-vsr-plugin` off `main`
- **PR sequence**:
  1. GIE dependency update + `pkg/bbr/vsr/types.go` + `vsr_client.go` + client tests
  2. `vsr_plugin.go` + plugin tests + `register.go`
  3. BBR binary integration (`cmd/bbr/`) + Dockerfile (follow-up)

---

## Related Components (for broader e2e)

| Component | Role | Repo |
|-----------|------|------|
| **IGW BBR** | Body-based routing with plugin chain | kubernetes-sigs/gateway-api-inference-extension |
| **vSR** | Semantic routing via /v1/route | vLLM Semantic Router |
| **IGW EPP / llm-d scheduler** | Endpoint picking (KV cache, load, P/D) | llm-d/llm-d-inference-scheduler |
| **MaaS** | Model deployment + access control | opendatahub-io/models-as-a-service |
| **Authorino** (Kuadrant) | Authentication/authorization (ext-authz) | Kuadrant/authorino |
| **Limitador** (Kuadrant) | Rate limiting (gRPC, Envoy RLS v3) | Kuadrant/limitador |
| **Istio** | Gateway + service mesh | istio/istio |
