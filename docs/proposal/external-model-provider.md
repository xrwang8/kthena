# Proposal: ExternalModelProvider for Third-Party LLM API Routing

## Summary

This proposal introduces `ExternalModelProvider`, a new CRD that enables Kthena Router to forward inference requests to external OpenAI-compatible LLM APIs (OpenAI, vLLM, and other OpenAI-compatible endpoints) while preserving existing in-cluster routing semantics.

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
- Preserve existing routing precedence: ModelRoute → HTTPRoute → ExternalModelProvider
- Support multiple providers for the same model name with weight-based load balancing
- Namespace isolation for multi-tenant deployments

### Non-Goals

- Support for non-OpenAI-compatible APIs (can be added later)
- Built-in API key rotation (users manage Secrets)
- Request/response transformation beyond model name rewriting
- Cost tracking or billing integration

## Proposal

### API Design

#### ExternalModelProvider CRD

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
    # Reference to Secret containing the API key (key name: apiKey)
    secretRef: openai-api-key
    # Optional: custom header name (default: "Authorization")
    headerName: ""
    # Optional: value prefix prepended before the secret value (default: "Bearer ")
    # Set to "" for providers that expect a raw token without a prefix
    # Examples:
    #   scheme: "Bearer "  → Authorization: Bearer <token>  (OpenAI default)
    #   scheme: ""         → api-key: <token>               (Azure OpenAI style)
    scheme: "Bearer "

  # Model mappings: client model name → upstream model name
  models:
    - name: gpt-4-mini          # Client requests this name
      upstreamModel: gpt-4o-mini # Provider receives this name
    - name: gpt-4
      upstreamModel: gpt-4-turbo

  # Optional: request timeout (default: 60s)
  timeout: 60s

  # Optional: weight for load balancing (default: 1)
  # weight is a provider-level field. When multiple providers map the same
  # client model name, traffic is distributed proportionally by weight.
  # This mirrors ModelRoute's TargetModel.Weight semantics.
  weight: 1

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

### Validation Rules

- `spec.baseURL`: required, must be a valid HTTP or HTTPS URL and must **not** end with a version path segment (e.g. `/v1`, `/v2`). The router appends the full inbound path as-is, so a trailing version segment would produce a doubled prefix like `/v1/v1/chat/completions`. CRD validation will enforce this with a rule equivalent to: `!self.matches('^.*/v\\d+/?$')`.
- `spec.models`: required, must contain at least one entry
- `spec.models[].name`: required, must be non-empty
- `spec.auth.secretRef`: required, must be non-empty; Secret must contain key `apiKey`
- `spec.auth.headerName`: optional, default `Authorization`
- `spec.auth.scheme`: optional, default `Bearer ` (note trailing space); set to `""` for raw token
- `spec.weight`: optional, range [0, 100], default 1
- `spec.timeout`: optional, must be a positive duration

### baseURL Path Construction

`baseURL` is treated as the upstream API root and is concatenated directly with the inbound request path. The router does **not** strip or rewrite any path segments.

| baseURL | Inbound path | Upstream URL |
|---------|-------------|--------------|
| `https://api.openai.com/v1` | `/v1/chat/completions` | `https://api.openai.com/v1/v1/chat/completions` ❌ |
| `https://api.openai.com` | `/v1/chat/completions` | `https://api.openai.com/v1/chat/completions` ✅ |

**Rule:** set `baseURL` to the API root without a trailing path version segment. The router appends the full inbound path (`/v1/chat/completions`) as-is.

### Routing Behavior

**Precedence Order:**
1. `ModelRoute` — In-cluster model serving endpoints
2. `HTTPRoute` — Custom HTTP routes
3. `ExternalModelProvider` — External APIs (fallback only)

**Path Filtering:**
- Only `POST /v1/chat/completions` triggers external routing in the initial delivery

**Multi-Provider Selection:**
- When multiple providers serve the same model name, traffic is split by weight
- Non-ready providers (missing Secret) are excluded from selection

**Namespace Isolation:**
- Router resolves providers only within the request's effective namespace
- Effective namespace: Gateway namespace if gatewayKey is present, otherwise `POD_NAMESPACE`

### Architecture

```
Client Request
      │
      ▼
┌─────────────────────────────────────────┐
│             Kthena Router               │
│                                         │
│  1. ModelRoute lookup                   │
│         │ miss                          │
│  2. HTTPRoute lookup                    │
│         │ miss                          │
│  3. ExternalModelProvider lookup        │
│     (POST /v1/chat/completions only)    │
│         │ miss                          │
│  4. 404 route not found                 │
└─────────────────────────────────────────┘
      │ matched
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

### Use Cases

#### 1. OpenAI Integration

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: openai
  namespace: default
spec:
  baseURL: https://api.openai.com
  auth:
    secretRef: openai-key
  models:
    - name: gpt-4-mini
      upstreamModel: gpt-4o-mini
```

#### 2. vLLM OpenAI-Compatible Endpoint

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: vllm-external
  namespace: default
spec:
  baseURL: https://my-vllm-service.example.com
  auth:
    secretRef: vllm-key
    scheme: "Bearer "
  models:
    - name: llama-3-70b
      upstreamModel: meta-llama/Meta-Llama-3-70B-Instruct
  timeout: 120s
```

> **Note on Azure OpenAI:** Azure OpenAI requires a deployment-specific URL path and a mandatory `api-version` query parameter. Supporting Azure requires adding a `queryParams` field to the CRD, which is deferred to a future iteration.

#### 3. Multi-Provider Load Balancing

```yaml
# 70% traffic to OpenAI
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: openai-primary
  namespace: default
spec:
  baseURL: https://api.openai.com
  auth:
    secretRef: openai-key
  models:
    - name: gpt-4-mini
      upstreamModel: gpt-4o-mini
  weight: 7
---
# 30% traffic to another OpenAI-compatible endpoint
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ExternalModelProvider
metadata:
  name: openai-backup
  namespace: default
spec:
  baseURL: https://backup-openai-proxy.example.com
  auth:
    secretRef: backup-key
  models:
    - name: gpt-4-mini
      upstreamModel: gpt-4o-mini
  weight: 3
```

## Alternatives Considered

### 1. Extend HTTPRoute with Auth
**Rejected:** HTTPRoute is generic and should not have LLM-specific features like model name rewriting.

### 2. Use Gateway API ExternalName Service
**Rejected:** Does not support Secret-based auth injection or model name rewriting.

### 3. Deploy Separate Proxy Service
**Rejected:** Adds operational overhead and duplicates router functionality.

## Open Questions

1. Should we support non-OpenAI-compatible APIs in a future iteration?
2. Should we support multiple `baseURL` entries per provider for regional failover? (Future work)

## References

- Issue: https://github.com/volcano-sh/kthena/issues/939
- OpenAI API Reference: https://platform.openai.com/docs/api-reference
- Azure OpenAI Service: https://learn.microsoft.com/en-us/azure/ai-services/openai/
