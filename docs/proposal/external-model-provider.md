# Proposal: ExternalModelProvider for Third-Party LLM API Routing

## Summary

This proposal introduces `ExternalModelProvider`, a new CRD that enables Kthena Router to forward inference requests to external OpenAI-compatible LLM APIs (OpenAI, vLLM, and other OpenAI-compatible endpoints) through `ModelRoute`, preserving existing routing semantics and traffic management capabilities.

## Motivation

Currently, Kthena Router only supports routing to in-cluster model serving endpoints via `ModelRoute` and `HTTPRoute`. Users who want to integrate external LLM APIs must:

1. Deploy custom proxy services in the cluster
2. Manually manage API keys and authentication
3. Implement model name mapping logic
4. Handle streaming responses correctly

This creates operational overhead and duplicates functionality that should be built into the router.

### Goals

- Enable routing to external LLM APIs without deploying additional proxy services
- Support OpenAI-compatible APIs (OpenAI, vLLM, and other OpenAI-compatible endpoints)
- Reuse `ModelRoute` for unified traffic management (weight-based splitting, rate limiting, etc.)
- Support mixed routing: in-cluster + external backends for the same model name
- Namespace isolation for multi-tenant deployments
- Provide extension points for cost-aware and latency-aware routing

### Non-Goals

- Support for non-OpenAI-compatible APIs (can be added later)
- Built-in API key rotation (users manage Secrets)
- Request/response transformation beyond model name rewriting
- Cost tracking or billing integration (extension point provided)

## Proposal

### API Design

#### ExternalModelProvider CRD

`ExternalModelProvider` defines an external API backend. It is referenced by `ModelRoute.TargetModel`, not queried independently.

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: openai-gpt4
  namespace: default
spec:
  # Base URL of the external API endpoint (do not include /v1 suffix)
  baseURL: https://api.openai.com

  # Authentication configuration
  auth:
    # Authentication type (default: Bearer)
    # Options: Bearer | APIKeyHeader | Raw
    type: Bearer
    
    # Reference to Secret containing the API key
    secretRef:
      name: openai-api-key
      key: apiKey              # Optional, defaults to "apiKey"
    
    # Custom header name (only used when type=APIKeyHeader)
    # Default: "Authorization" for Bearer, must be specified for APIKeyHeader
    headerName: api-key

  # Models provided by this external API
  # These are the upstream model names that the provider actually serves
  models:
    - name: gpt-4o-mini
    - name: gpt-4-turbo

  # Optional: request timeout (default: 60s)
  timeout: 60s

  # Optional: cost metadata for cost-aware routing (future extension)
  # +optional
  cost:
    perInputToken: 0.00015    # USD per 1K tokens
    perOutputToken: 0.00060

  # Optional: latency SLO for latency-aware routing (future extension)
  # +optional
  latency:
    p50Ms: 120
    p99Ms: 350

status:
  conditions:
    - type: Ready
      status: "True"
      reason: ProviderReady
      message: "Provider is ready"
    - type: SecretAvailable
      status: "True"
      reason: SecretFound
      message: "Secret openai-api-key found"
```

#### Secret Format

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openai-api-key
  namespace: default
type: Opaque
stringData:
  apiKey: sk-...
```

#### ModelRoute Integration

`ExternalModelProvider` is referenced by `ModelRoute.TargetModel`, enabling unified traffic management:

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ModelRoute
metadata:
  name: gpt-4-mini-route
  namespace: default
spec:
  # Client-facing model name
  modelName: gpt-4-mini
  
  # Traffic targets: mix in-cluster and external backends
  targetModels:
    # 80% traffic to in-cluster ModelServer
    - modelServerName: local-llama
      weight: 80
    
    # 20% traffic to external OpenAI API
    - externalProviderRef:
        name: openai-gpt4
        modelName: gpt-4o-mini    # Which model in the provider
      weight: 20
```

#### TargetModel API Changes

`TargetModel` is extended to support external providers via a discriminated union:

```go
type TargetModel struct {
    // Exactly one of the following must be set:
    
    // ModelServerName references an in-cluster ModelServer
    // +optional
    ModelServerName string `json:"modelServerName,omitempty"`
    
    // ExternalProviderRef references an ExternalModelProvider
    // +optional
    ExternalProviderRef *ExternalProviderRef `json:"externalProviderRef,omitempty"`
    
    // Weight for traffic splitting (0-100)
    // +optional
    // +kubebuilder:default=100
    // +kubebuilder:validation:Minimum=0
    // +kubebuilder:validation:Maximum=100
    Weight *uint32 `json:"weight,omitempty"`
}
// +kubebuilder:validation:XValidation:rule="has(self.modelServerName) != has(self.externalProviderRef)",message="exactly one of modelServerName or externalProviderRef must be set"

type ExternalProviderRef struct {
    // Name of the ExternalModelProvider in the same namespace
    // +kubebuilder:validation:required
    Name string `json:"name"`
    
    // ModelName specifies which model in the provider to use
    // Must match one of ExternalModelProvider.spec.models[].name
    // +kubebuilder:validation:required
    ModelName string `json:"modelName"`
}
```

**CRD Validation**: The CEL rule `has(self.modelServerName) != has(self.externalProviderRef)` ensures exactly one field is set.

### Validation Rules

**ExternalModelProvider**:
- `spec.baseURL`: required, must be a valid HTTP or HTTPS URL and must **not** end with a version path segment (e.g. `/v1`, `/v2`). The router appends the full inbound path as-is, so a trailing version segment would produce a doubled prefix like `/v1/v1/chat/completions`. CRD validation enforces this with: `!self.matches('^.*/v\\d+/?$')`.
- `spec.models`: required, must contain at least one entry
- `spec.models[].name`: required, must be non-empty
- `spec.auth.secretRef.name`: required, must be non-empty
- `spec.auth.secretRef.key`: optional, default `apiKey`
- `spec.auth.type`: optional, default `Bearer`; valid values: `Bearer`, `APIKeyHeader`, `Raw`
- `spec.auth.headerName`: required when `type=APIKeyHeader`; ignored otherwise
- `spec.timeout`: optional, must be a positive duration

**TargetModel**:
- Exactly one of `modelServerName` or `externalProviderRef` must be set
- `externalProviderRef.name`: required when `externalProviderRef` is set
- `externalProviderRef.modelName`: required when `externalProviderRef` is set

### baseURL Path Construction

`baseURL` is treated as the upstream API root and is concatenated directly with the inbound request path. The router does **not** strip or rewrite any path segments.

| baseURL | Inbound path | Upstream URL |
|---------|-------------|--------------|
| `https://api.openai.com/v1` | `/v1/chat/completions` | `https://api.openai.com/v1/v1/chat/completions` ❌ |
| `https://api.openai.com` | `/v1/chat/completions` | `https://api.openai.com/v1/chat/completions` ✅ |

**Rule:** set `baseURL` to the API root without a trailing path version segment. The router appends the full inbound path (`/v1/chat/completions`) as-is.

### Routing Behavior

**Path Filtering:**
- Only `POST /v1/chat/completions` triggers external routing in the initial delivery

**Namespace Isolation:**
- Router resolves `ExternalModelProvider` only within the request's effective namespace
- Effective namespace: Gateway namespace if gatewayKey is present, otherwise `POD_NAMESPACE`

**Model Name Mapping:**
- Client sends `model: gpt-4-mini` (in request body)
- Router looks up `ModelRoute` with `modelName: gpt-4-mini`
- Router selects a `TargetModel` (by weight or future selection policy)
- If selected target is `externalProviderRef`, router rewrites request body to `model: <externalProviderRef.modelName>`
- Router forwards to `ExternalModelProvider.spec.baseURL`

### Architecture

```
Client Request (model: gpt-4-mini)
      │
      ▼
┌─────────────────────────────────────────┐
│             Kthena Router               │
│                                         │
│  1. Lookup ModelRoute (modelName)      │
│  2. Select TargetModel (by weight)     │
│     ├─ modelServerName?                │
│     │  └─ Forward to in-cluster        │
│     └─ externalProviderRef?            │
│        ├─ Resolve ExternalModelProvider│
│        ├─ Rewrite model field          │
│        └─ Inject auth header           │
└─────────────────────────────────────────┘
      │ externalProviderRef
      ▼
┌─────────────────────────────────────────┐
│           External Proxy                │
│  - Rewrite model field                  │
│  - Inject auth header from Secret       │
│  - Forward to provider baseURL          │
│  - Stream response back to client       │
└─────────────────────────────────────────┘
      │
      ▼
External API (OpenAI, Azure OpenAI, etc.)
```

## Design Rationale & Review Response

This section addresses review feedback from @hzxuzhonghu (PR #968).

### 1. Cost-Aware Routing Extension Point

**Review**: "Consider how can we expand the api to support cost-saving model routing"

**Response**: The new design provides a clear extension point for cost-aware routing:

- **Current**: `ModelRoute` selects `TargetModel` by weight (default behavior)
- **Future**: Add `selectionPolicy` to `ModelRouteSpec`:
  ```yaml
  spec:
    modelName: gpt-4-mini
    selectionPolicy: LeastCost  # or Weighted (default), LowestLatency
    targetModels: [...]
  ```
- **Provider metadata**: Add optional `cost` and `latency` fields to `ExternalModelProvider.spec`:
  ```yaml
  spec:
    cost:
      perInputToken: 0.00015   # USD per 1K tokens
      perOutputToken: 0.00060
    latency:
      p50Ms: 120
      p99Ms: 350
  ```

When `selectionPolicy=LeastCost`, router estimates request cost based on token count and selects the cheapest available target (in-cluster or external). This works uniformly for both `ModelServer` and `ExternalModelProvider` targets.

**Key benefit**: No need for separate selection logic for external providers. Cost-aware routing is a `ModelRoute` capability that applies to all target types.

### 2. Customizable Secret Key

**Review**: "what `key` of the secret? Should we allow customize?"

**Response**: Changed `secretRef` from a string to a structured reference:

```yaml
auth:
  secretRef:
    name: openai-api-key
    key: apiKey          # Optional, defaults to "apiKey"
```

This follows standard Kubernetes patterns (e.g., `envFrom.secretKeyRef`) and allows users to reuse existing Secrets without creating new ones.

### 3. Bearer Token as Default

**Review**: "Most maas requires bearer token"

**Response**: Simplified `auth` to use an enum with `Bearer` as the default:

```yaml
auth:
  type: Bearer           # Default; other options: APIKeyHeader, Raw
  secretRef: { name, key }
  headerName: api-key    # Only used when type=APIKeyHeader
```

**Behavior**:
- `type: Bearer` (default) → `Authorization: Bearer <token>`
- `type: APIKeyHeader` → `<headerName>: <token>` (e.g., `api-key: <token>` for Azure)
- `type: Raw` → `Authorization: <token>` (no "Bearer " prefix)

90% of users get the right behavior with zero configuration.

### 4. Relationship with ModelRoute

**Review**: "Explain how this works? We have model defined in modelRoute, so this need to match that one?"

**Response**: The new design eliminates this confusion by making `ExternalModelProvider` a **backend reference**, not an independent routing layer.

**How it works**:
1. `ModelRoute.spec.modelName` defines the client-facing model name
2. `ModelRoute.spec.targetModels[]` defines where traffic goes (in-cluster or external)
3. `TargetModel.externalProviderRef.modelName` specifies which upstream model to use

**Example**:
```yaml
kind: ModelRoute
spec:
  modelName: gpt-4-mini              # Client sees this
  targetModels:
    - externalProviderRef:
        name: openai-gpt4
        modelName: gpt-4o-mini       # Provider receives this
      weight: 100
```

**No ambiguity**: There is no "matching" between `ModelRoute` and `ExternalModelProvider`. The relationship is explicit via `externalProviderRef`.

**Mixed routing**: Naturally supported by adding multiple `TargetModel` entries:
```yaml
targetModels:
  - modelServerName: local-llama     # 80% in-cluster
    weight: 80
  - externalProviderRef:             # 20% external
      name: openai-gpt4
      modelName: gpt-4o-mini
    weight: 20
```

## Use Cases

### 1. OpenAI Integration

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: openai
  namespace: default
spec:
  baseURL: https://api.openai.com
  auth:
    type: Bearer
    secretRef:
      name: openai-key
  models:
    - name: gpt-4o-mini
    - name: gpt-4-turbo
---
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ModelRoute
metadata:
  name: gpt-4-mini-route
  namespace: default
spec:
  modelName: gpt-4-mini
  targetModels:
    - externalProviderRef:
        name: openai
        modelName: gpt-4o-mini
      weight: 100
```

### 2. vLLM OpenAI-Compatible Endpoint

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: vllm-external
  namespace: default
spec:
  baseURL: https://my-vllm-service.example.com
  auth:
    type: Bearer
    secretRef:
      name: vllm-key
  models:
    - name: meta-llama/Meta-Llama-3-70B-Instruct
  timeout: 120s
---
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ModelRoute
metadata:
  name: llama-3-70b-route
  namespace: default
spec:
  modelName: llama-3-70b
  targetModels:
    - externalProviderRef:
        name: vllm-external
        modelName: meta-llama/Meta-Llama-3-70B-Instruct
      weight: 100
```

### 3. Multi-Provider Load Balancing

```yaml
# Primary OpenAI provider (70% traffic)
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: openai-primary
  namespace: default
spec:
  baseURL: https://api.openai.com
  auth:
    type: Bearer
    secretRef:
      name: openai-key
  models:
    - name: gpt-4o-mini
---
# Backup provider (30% traffic)
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: openai-backup
  namespace: default
spec:
  baseURL: https://backup-openai-proxy.example.com
  auth:
    type: Bearer
    secretRef:
      name: backup-key
  models:
    - name: gpt-4o-mini
---
# ModelRoute with traffic splitting
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ModelRoute
metadata:
  name: gpt-4-mini-route
  namespace: default
spec:
  modelName: gpt-4-mini
  targetModels:
    - externalProviderRef:
        name: openai-primary
        modelName: gpt-4o-mini
      weight: 70
    - externalProviderRef:
        name: openai-backup
        modelName: gpt-4o-mini
      weight: 30
```

### 4. Hybrid Routing (In-Cluster + External)

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ModelRoute
metadata:
  name: gpt-4-mini-hybrid
  namespace: default
spec:
  modelName: gpt-4-mini
  targetModels:
    # 80% to in-cluster ModelServer
    - modelServerName: local-llama-server
      weight: 80
    # 20% to external OpenAI (overflow/canary)
    - externalProviderRef:
        name: openai
        modelName: gpt-4o-mini
      weight: 20
```

> **Note on Azure OpenAI:** Azure OpenAI requires a deployment-specific URL path and a mandatory `api-version` query parameter. Supporting Azure requires adding a `queryParams` field to the CRD, which is deferred to a future iteration.

## Extension Points

This proposal implements weight-based traffic splitting. The following extension points are designed into the API for future enhancements without breaking changes:

### 1. Cost-Aware Routing

Add optional cost metadata to `ExternalModelProvider`:

```yaml
spec:
  cost:
    perInputToken: 0.00015    # USD per 1K tokens
    perOutputToken: 0.00060
```

Add `selectionPolicy` to `ModelRouteSpec`:

```yaml
spec:
  modelName: gpt-4-mini
  selectionPolicy: LeastCost  # Options: Weighted (default), LeastCost, LowestLatency
  targetModels: [...]
```

When `selectionPolicy=LeastCost`, router estimates request cost based on token count and selects the cheapest available target. This works uniformly for both in-cluster and external targets.

### 2. Latency-Aware Routing

Add latency metadata to `ExternalModelProvider.status` (observed) or `spec` (declared SLO):

```yaml
status:
  observedLatency:
    p50Ms: 120
    p99Ms: 350
```

When `selectionPolicy=LowestLatency`, router selects the target with the best latency profile.

### 3. Quota and Rate-Limit Aware Routing

When an external provider returns HTTP 429 (rate limit exceeded), router can:
- Temporarily exclude it from selection
- Automatically failover to other providers serving the same model
- Emit metrics for alerting

### 4. Regional Failover

Add multiple `baseURL` entries per provider for regional redundancy:

```yaml
spec:
  baseURLs:
    - url: https://api.openai.com
      region: us-east-1
      priority: 1
    - url: https://eu-api.openai.com
      region: eu-west-1
      priority: 2
```

Router tries URLs in priority order, failing over on connection errors.

## Alternatives Considered

### 1. Independent Fallback Layer (Original Design)

**Rejected**: The original design had `ExternalModelProvider` as an independent routing layer with its own weight-based selection, separate from `ModelRoute`. This created:
- Two separate weight mechanisms (ModelRoute vs ExternalProvider)
- Confusion about the relationship between ModelRoute and ExternalProvider
- No path for mixed routing (in-cluster + external)
- Duplicate selection logic for future cost-aware routing

The new design unifies all routing through `ModelRoute`, making `ExternalModelProvider` a backend reference like `ModelServer`.

### 2. Extend HTTPRoute with Auth

**Rejected**: HTTPRoute is generic and should not have LLM-specific features like model name rewriting and streaming response handling.

### 3. Use Gateway API ExternalName Service

**Rejected**: Does not support Secret-based auth injection or model name rewriting.

### 4. Deploy Separate Proxy Service

**Rejected**: Adds operational overhead and duplicates router functionality.

## Questions

1. Should we support non-OpenAI-compatible APIs in a future iteration? (e.g., Anthropic Claude API, Google Gemini API)
2. Should we support Azure OpenAI's deployment-specific URL patterns and query parameters in the initial delivery or defer to a future iteration?
