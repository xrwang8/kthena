---
title: ExternalModelProvider for Third-Party LLM API Routing
authors:
- "@xrwang8"
reviewers:
- "@hzxuzhonghu"
- "@YaoZengzeng"
- "@LiZhenCheng9527"
- "@git-malu"
- TBD
approvers:
- "@LiZhenCheng9527"
- TBD

creation-date: 2026-05-09

---

## ExternalModelProvider for Third-Party LLM API Routing

### Summary

This proposal introduces `ExternalModelProvider`, a new CRD that enables Kthena Router to forward inference requests to external OpenAI-compatible LLM APIs (OpenAI, vLLM, and other OpenAI-compatible endpoints) through `ModelRoute`, preserving existing routing semantics and traffic management capabilities.

### Motivation

Currently, Kthena Router only supports routing to in-cluster model serving endpoints via `ModelRoute` and `HTTPRoute`. Users who want to integrate external LLM APIs must:

1. Deploy custom proxy services in the cluster
2. Manually manage API keys and authentication
3. Implement model name mapping logic
4. Handle streaming responses correctly

This creates operational overhead and duplicates functionality that should be built into the router.

#### Goals

- Enable routing to external LLM APIs without deploying additional proxy services
- Support OpenAI-compatible APIs (OpenAI, vLLM, and other OpenAI-compatible endpoints)
- Reuse `ModelRoute` for unified traffic management (weight-based splitting, rate limiting, etc.)
- Support mixed routing: in-cluster + external backends for the same model name
- Namespace isolation for multi-tenant deployments
- Reserve a forward-compatible extension point on `ModelRoute` (`selectionPolicy`) for future cost-aware and latency-aware routing, without committing to a cost-data schema in v1alpha1

#### Non-Goals

- Support for non-OpenAI-compatible APIs (can be added later)
- Built-in API key rotation (users manage Secrets)
- Request/response transformation beyond model name rewriting
- Cost / billing data modeling on `ExternalModelProvider` in v1alpha1 — deferred to a follow-up proposal that selects the data source (see Design Rationale §1)

### Proposal

#### API Design

#### ExternalModelProvider CRD

`ExternalModelProvider` defines an external API backend. It is referenced by `ModelRoute.rules[].targetModels[]`, not queried independently.

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
    # Options: Bearer | APIKeyHeader | RawToken
    type: Bearer
    
    # Reference to Secret containing the API key.
    # Mirrors corev1.SecretKeySelector while defaulting key to "apiKey".
    secretRef:
      name: openai-api-key
      key: apiKey              # Optional, defaults to "apiKey"

    # headerName is omitted for Bearer and RawToken.
    # For type=APIKeyHeader, set the custom header name, e.g. "api-key".

  # Models provided by this external API
  # These are the upstream model names that the provider actually serves
  models:
    - name: gpt-4o-mini
    - name: gpt-4-turbo

  # Optional: request timeout (default: 60s)
  timeout: 60s

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

`secretRef` intentionally uses a small local struct instead of embedding
`corev1.SecretKeySelector` directly, because this API wants `key` to be optional
and default to `apiKey`.

```go
type ExternalSecretKeyRef struct {
    // Name of the Secret in the same namespace as the ExternalModelProvider.
    // +kubebuilder:validation:Required
    Name string `json:"name"`

    // Key in the Secret data map. Defaults to "apiKey".
    // +optional
    // +kubebuilder:default=apiKey
    Key string `json:"key,omitempty"`
}

// AuthType defines the authentication scheme used when forwarding requests.
// +kubebuilder:validation:Enum=Bearer;APIKeyHeader;RawToken
type AuthType string

const (
    // AuthTypeBearer injects "Authorization: Bearer <token>".
    AuthTypeBearer AuthType = "Bearer"
    // AuthTypeAPIKeyHeader injects "<headerName>: <token>".
    AuthTypeAPIKeyHeader AuthType = "APIKeyHeader"
    // AuthTypeRawToken injects "Authorization: <token>" with no scheme prefix.
    AuthTypeRawToken AuthType = "RawToken"
)

// +kubebuilder:validation:XValidation:rule="(self.type == 'APIKeyHeader') == has(self.headerName)",message="headerName is required when type=APIKeyHeader and must be omitted otherwise"
type ExternalModelProviderAuth struct {
    // Type of authentication scheme. Defaults to Bearer.
    // +optional
    // +kubebuilder:default=Bearer
    Type AuthType `json:"type,omitempty"`

    // SecretRef references the Secret containing the API key.
    // +kubebuilder:validation:Required
    SecretRef ExternalSecretKeyRef `json:"secretRef"`

    // HeaderName is the HTTP header used to carry the token.
    // Required when type=APIKeyHeader; rejected otherwise.
    // +optional
    HeaderName string `json:"headerName,omitempty"`
}
```

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

The full Go type for `ExternalModelProvider` (spec and status):

```go
type ExternalModelProviderSpec struct {
    // BaseURL is the HTTPS root of the external API (no trailing /v1).
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:XValidation:rule="self.lowerAscii().startsWith('https://')",message="baseURL must use HTTPS"
    // +kubebuilder:validation:XValidation:rule="!self.contains('@')",message="baseURL must not contain userinfo"
    // +kubebuilder:validation:XValidation:rule="!self.matches('^.*/v\\\\d+/?$')",message="baseURL must not end with a version path segment"
    BaseURL string `json:"baseURL"`

    // Auth configures how the router authenticates to the external API.
    // +kubebuilder:validation:Required
    Auth ExternalModelProviderAuth `json:"auth"`

    // Models lists the upstream model names this provider serves.
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=64
    Models []ExternalModel `json:"models"`

    // Timeout for requests to this provider. Defaults to 60s.
    // Applies as an idle timeout for streaming responses.
    // +optional
    Timeout *metav1.Duration `json:"timeout,omitempty"`
}

type ExternalModel struct {
    // Name is the upstream model identifier (e.g. "gpt-4o-mini").
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MaxLength=253
    Name string `json:"name"`
}

// ExternalModelProviderStatus defines the observed state of ExternalModelProvider.
type ExternalModelProviderStatus struct {
    // ObservedGeneration is the .metadata.generation the controller last reconciled.
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions summarize the provider's readiness.
    // Known types: Ready, SecretAvailable.
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```



`ExternalModelProvider` is referenced by `ModelRoute.rules[].targetModels`, enabling unified traffic management:

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ModelRoute
metadata:
  name: gpt-4-mini-route
  namespace: default
spec:
  # Client-facing model name
  modelName: gpt-4-mini
  rules:
    - name: default
      # Empty modelMatch matches requests selected by spec.modelName.
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
// +kubebuilder:validation:XValidation:rule="[has(self.modelServerName), has(self.externalProviderRef)].filter(x, x).size() == 1",message="exactly one of modelServerName or externalProviderRef must be set"
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

type ExternalProviderRef struct {
    // Name of the ExternalModelProvider in the same namespace
    // +kubebuilder:validation:Required
    Name string `json:"name"`
    
    // ModelName specifies which model in the provider to use
    // Must match one of ExternalModelProvider.spec.models[].name
    // +kubebuilder:validation:Required
    ModelName string `json:"modelName"`
}
```

**CRD Validation**: The CEL rule uses `filter` form rather than `!=` to ensure it extends cleanly when a third backend type is added: `[has(self.modelServerName), has(self.externalProviderRef)].filter(x, x).size() == 1`. Adding a third field (e.g. `inferencePoolRef`) only requires appending it to the list — no rule replacement needed.

#### ModelRouteSpec Extension: SelectionPolicy

`ModelRouteSpec` is extended with an optional `selectionPolicy` field. v1alpha1
only defines `Weighted` (the current behavior). Future policy types will be
added in follow-up proposals; admitting an unknown value at the API server is
not the right way to "reserve" them, because it forces consumers to handle
silent fallback.

```go
type ModelRouteSpec struct {
    ModelName string `json:"modelName,omitempty"`
    Rules     []*Rule `json:"rules"`

    // SelectionPolicy determines how to pick a target when multiple are eligible.
    // If unset, Weighted is used (current behavior, backward compatible).
    // +optional
    SelectionPolicy *SelectionPolicy `json:"selectionPolicy,omitempty"`
}

type Rule struct {
    // Match conditions to be satisfied for the rule to be activated.
    // Empty modelMatch means matching all requests.
    // +optional
    ModelMatch *ModelMatch `json:"modelMatch,omitempty"`

    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=16
    TargetModels []*TargetModel `json:"targetModels"`
}

type SelectionPolicy struct {
    // Type of selection strategy.
    // v1alpha1 only accepts "Weighted". Additional types (e.g. LeastCost,
    // LowestLatency) will be introduced in follow-up proposals when their
    // data sources and semantics are defined.
    // +kubebuilder:default=Weighted
    Type SelectionPolicyType `json:"type"`
}

// SelectionPolicyType enumerates the supported target-selection strategies.
// +kubebuilder:validation:Enum=Weighted
type SelectionPolicyType string

const (
    // SelectionPolicyWeighted picks a target proportionally to TargetModel.Weight.
    SelectionPolicyWeighted SelectionPolicyType = "Weighted"
)
```

**Why a struct, not a plain enum field**: future policies will need parameters
(e.g. cost budget, latency SLO, fallback chain). Wrapping the enum in a struct
keeps the API forward-compatible without breaking changes — new policies are
added by appending to the enum and adding sibling fields on `SelectionPolicy`,
never by mutating existing field shapes.

**Backward compatibility of `TargetModel`**: existing v1alpha1 `ModelRoute`
resources that set only `modelServerName` remain valid — the new CEL rule
accepts them unchanged. No data migration is required.

#### ModelRouteStatus Extension: Conditions

The current `ModelRouteStatus` is empty. This proposal adds a `Conditions`
field so the `ExternalModelProvider` controller can surface cross-resource
validation failures (e.g. a `TargetModel.externalProviderRef.modelName` that
does not exist in the referenced provider's `models[]`).

```go
// ModelRouteStatus defines the observed state of ModelRoute.
type ModelRouteStatus struct {
    // ObservedGeneration is the .metadata.generation the controller last reconciled.
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions summarize the route's readiness and any cross-resource
    // validation failures. Known types:
    //   - Ready: True when all references resolve.
    //   - ModelNotFoundInProvider: True when an externalProviderRef.modelName
    //     does not match any entry in the referenced provider's spec.models[].
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

**Backward compatibility**: adding fields to a previously empty `Status` is
non-breaking. Existing controllers that ignore `status` continue to work; new
consumers can opt into reading `conditions`.

One edge case: the current `modelroute_types.go:110` marks `ModelServerName`
as `+kubebuilder:validation:required`, which enforces key presence but not
non-empty value. A stored object with `modelServerName: ""` would have
`has(self.modelServerName) == false` under the new schema (because the field
becomes `omitempty`), causing the CEL rule to reject it on the next update.
Mitigation: add `+kubebuilder:validation:MinLength=1` to `ModelServerName`
in the same PR that removes `required`. This was always the correct constraint;
the migration is safe because an empty `modelServerName` was never a valid
reference to an in-cluster `ModelServer`.

#### Validation Rules

**ExternalModelProvider**:
- `spec.baseURL`: required, must be a valid **HTTPS** URL. HTTP is rejected because auth headers (Bearer token / API key) are always injected; allowing HTTP would leak credentials to any network observer on the path. CRD validation enforces this with two rules:
  1. `self.lowerAscii().startsWith('https://')` — rejects HTTP and other schemes.
  2. `!self.contains('@')` — rejects userinfo-form URLs (e.g. `https://token@evil.example/`) that could redirect auth headers to an attacker-controlled endpoint (SSRF via credential forwarding).
- `spec.baseURL`: must **not** end with a version path segment (e.g. `/v1`, `/v2`). The router appends the full inbound path as-is, so a trailing version segment would produce a doubled prefix like `/v1/v1/chat/completions`. CRD validation enforces this with: `!self.matches('^.*/v\\d+/?$')`.
- `spec.models`: required, must contain at least one entry
- `spec.models[].name`: required, must be non-empty
- `spec.auth.secretRef`: required; uses `ExternalSecretKeyRef`, a `corev1.SecretKeySelector`-shaped struct with `name` required and `key` optional/defaulted to `apiKey`
- `spec.auth.type`: optional, default `Bearer`; valid values: `Bearer`, `APIKeyHeader`, `RawToken`
- `spec.auth.headerName`: **required** when `type=APIKeyHeader`; **rejected** otherwise. Enforced by CEL: `(self.type == 'APIKeyHeader') == has(self.headerName)`. This converts the previous "silently ignored" behavior into a clear admission error.
- `spec.timeout`: optional, must be a positive duration

**TargetModel**:
- Exactly one of `modelServerName` or `externalProviderRef` must be set (CEL filter rule on `TargetModel`)
- `modelServerName`: when set, must be non-empty (`+kubebuilder:validation:MinLength=1`). Replaces the prior `+kubebuilder:validation:required` on `modelroute_types.go:110` — see the edge-case paragraph above.
- `externalProviderRef.name`: required when `externalProviderRef` is set
- `externalProviderRef.modelName`: required when `externalProviderRef` is set
- **Cross-resource consistency** (`externalProviderRef.modelName` must match a name in `ExternalModelProvider.spec.models[]`) is **not** enforced by CRD validation, since CEL cannot read other objects. The `ExternalModelProvider` controller emits a Warning event on referencing `ModelRoute` resources when it detects an invalid model reference; the router data plane returns a clear error if a request hits an invalid reference.

**ModelRouteSpec**:
- `selectionPolicy.type`: optional, default `Weighted`. v1alpha1 accepts only `Weighted`; future policy types will be added in follow-up proposals.

#### baseURL Path Construction

`baseURL` is treated as the upstream API root and is concatenated directly with the inbound request path. The router does **not** strip or rewrite any path segments.

| baseURL | Inbound path | Upstream URL |
|---------|-------------|--------------|
| `https://api.openai.com/v1` | `/v1/chat/completions` | `https://api.openai.com/v1/v1/chat/completions` ❌ |
| `https://api.openai.com` | `/v1/chat/completions` | `https://api.openai.com/v1/chat/completions` ✅ |

**Rule:** set `baseURL` to the API root without a trailing path version segment. The router appends the full inbound path (`/v1/chat/completions`) as-is.

#### Routing Behavior

**Path Filtering:**
- Only `POST /v1/chat/completions` triggers external routing in the initial delivery

**Namespace Isolation:**
- Router resolves `ExternalModelProvider` in the matched `ModelRoute` namespace
- The referenced Secret must be in the same namespace as the `ExternalModelProvider`
- Cross-namespace references are not supported in v1alpha1

**Model Name Mapping:**
- Client sends `model: gpt-4-mini` (in request body)
- Router looks up `ModelRoute` with `modelName: gpt-4-mini`
- Router selects a `TargetModel` (by weight or future selection policy)
- If selected target is `externalProviderRef`, router rewrites request body to `model: <externalProviderRef.modelName>`
- Router forwards to `ExternalModelProvider.spec.baseURL`

#### Controller Ownership & Status

A dedicated `ExternalModelProvider` controller owns the `status` subresource of
the CRD. Its responsibilities:

- **Secret watch**: watches the referenced `Secret` in the same namespace and
  updates `status.conditions[type=SecretAvailable]` when the Secret appears,
  disappears, or is missing the referenced key. The router data plane reads
  credentials directly from the informer cache; there is no controller-mediated
  credential plumbing, so Secret rotation is reflected on the next request
  without restart.
- **Reference validation**: watches `ModelRoute` resources that point at this
  provider and emits a Warning event on those routes
  when `externalProviderRef.modelName` does not match any entry in
  `spec.models[].name`. This is the cross-resource check that CRD CEL
  validation cannot express.
- **Ready aggregation**: `status.conditions[type=Ready]` is `True` iff the
  provider spec is valid and the referenced Secret is available. Invalid
  `ModelRoute` references do not make the provider itself NotReady.

The router data plane does **not** write to status; status is strictly
controller-owned.

#### Security Considerations

**Transport security**

`spec.baseURL` is restricted to HTTPS. Because the router unconditionally injects
the auth header (Bearer token or API key from the referenced Secret) into every
forwarded request, allowing plaintext HTTP would expose credentials to any
network observer or intermediary. There is no opt-in for HTTP in v1alpha1; if a
trusted in-cluster plaintext endpoint becomes necessary, a future iteration may
add an explicit, audit-friendly opt-in field.

**Same-namespace references only (v1alpha1)**

In v1alpha1, `ModelRoute`, `ExternalModelProvider`, and the referenced auth
`Secret` MUST reside in the same namespace. Cross-namespace references are not
supported. This matches the existing `ModelRoute` ↔ `ModelServer` relationship
and prevents a "confused deputy" scenario where a `ModelRoute` author could
indirectly consume a `Secret` they cannot read by referencing an
`ExternalModelProvider` owned by another tenant.

**Residual in-namespace risk (acknowledged)**

Same-namespace isolation does not eliminate confused-deputy entirely. If RBAC
within a namespace is fine-grained — for example, User A has `create` on
`ModelRoute` but no `get` on `Secret` — User A can still consume a Secret
indirectly by referencing an `ExternalModelProvider` they did not author. This
is the intended trade-off in v1alpha1; tightening it is the goal of the
authorization model below.

**Authorization model: future work**

A multi-tenant deployment in which `ExternalModelProvider` / `Secret` ownership
is separated from `ModelRoute` authorship requires explicit binding
authorization. Candidate mechanisms to be evaluated in a follow-up proposal:

- A `ReferenceGrant`-style cross-namespace permission resource (Gateway API
  pattern), allowing the provider's namespace to opt in to specific consumer
  namespaces or routes.
- An `allowedRoutes` selector on `ExternalModelProvider.spec`, listing which
  routes (by namespace/label) may reference it.
- An admission webhook that verifies the `ModelRoute` author has `get`
  permission on both the referenced `ExternalModelProvider` and its `Secret`.

This is deferred until a concrete multi-tenant use case emerges, to avoid
locking in a model before requirements are understood.

#### Architecture

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
│        ├─ Inject auth header           │
│        ├─ Forward to provider baseURL  │
│        └─ Stream response back         │
└─────────────────────────────────────────┘
      │ externalProviderRef
      ▼
External OpenAI-compatible API
```

The router performs auth injection, model field rewrite, and streaming proxy
in-process. There is no separate "external proxy" component to deploy.

### Design Rationale & Review Response

This section addresses review feedback from @hzxuzhonghu (PR #968).

#### 1. Cost-Aware Routing Extension Point

**Review**: "Consider how can we expand the api to support cost-saving model routing"

**Response**: This proposal intentionally does **not** add concrete cost fields
to the API. Instead, it reserves a clean extension point on `ModelRoute` so that
cost-aware routing can be added in a follow-up without breaking changes.

**What ships in this proposal**

- `ModelRoute.spec.selectionPolicy` (struct, not a plain enum field) is added.
  v1alpha1 accepts only `Weighted`; additional policy types will be introduced
  as the enum is widened in follow-up proposals, once their data sources and
  semantics are defined.
- `TargetModel` is already a unified abstraction over in-cluster (`ModelServerName`)
  and external (`ExternalProviderRef`) backends, so a future cost-aware selector
  can apply the same logic to both target types — no parallel mechanism for
  external providers.

**Why no `cost` field on `ExternalModelProvider`**

- **Vendor prices are external facts, not desired state.** OpenAI / Anthropic
  adjust prices every few months; encoding them in a CRD makes `kubectl apply`
  the de-facto price-update channel, with no clear ownership or freshness
  guarantee.
- **Premature schema lock-in.** Once a `cost` field is shipped, its units,
  granularity (per-provider vs per-model), and update path become contract.
  Every mainstream LLM gateway (LiteLLM, Portkey, OpenRouter, Envoy AI Gateway)
  keeps pricing data outside deployment resources for the same reason.

**Cost data source: deferred to a follow-up proposal**

The trade-offs below will be evaluated separately, once concrete user demand
clarifies which approach fits best:

| Approach | Pros | Cons |
|---|---|---|
| Per-model annotation on `ExternalModelProvider.models[]` | Simple, co-located | Manual upkeep, easily stale |
| Separate `ModelPriceCatalog` CRD with periodic sync | Decoupled, versionable, single source of truth | New resource to manage |
| External pricing service (LiteLLM-style table) | Real-time, industry-standard | Runtime dependency |
| Observed cost from request metrics | Data-driven, self-healing | Cold-start fallback needed |

Whichever data source is selected must provide comparable cost metadata for both
`modelServerName` and `externalProviderRef` targets; otherwise hybrid routing
cannot make a meaningful cost-saving decision.

**Cost attribution vs cost routing (two distinct concerns)**

Even after a follow-up proposal lands the data source, two cost concepts must
not be conflated:

| Concern | Dimension | What it answers |
|---|---|---|
| **Attribution** (billing, usage reports, quota) | `ModelRoute.spec.modelName` (the client-facing name, e.g. `gpt5`) | "How much did the user spend on `gpt5`?" |
| **Routing** (selector decision, internal) | per-target cost (per `ExternalModelProvider.models[]` entry, or per in-cluster target via a separate mechanism) | "Among the eligible backends, which is cheapest right now?" |

Concretely, given:

```yaml
kind: ModelRoute
spec:
  modelName: gpt5
  rules:
    - targetModels:
        - modelServerName: local-gpt5            # in-cluster
        - externalProviderRef:
            name: openai
            modelName: gpt-5                     # upstream
        - externalProviderRef:
            name: azure
            modelName: gpt-5                     # upstream
```

- **Cost / usage metrics MUST be tagged with `model="gpt5"`** (the
  `ModelRoute.modelName`), not the rewritten upstream name. Per-backend
  breakdown is exposed as a secondary label (e.g. `backend="openai-gpt-5"`)
  for internal cost analysis but does not replace the primary attribution.
- **Selector input is per-target cost**, not the aggregated `gpt5` cost. A
  single `gpt5` request may incur very different upstream costs depending on
  which backend handled it.

This separation is also why `cost` does not belong on `ModelRoute` itself: the
route describes *what is exposed to the client*; per-target cost describes
*what each backend charges*. Mixing them on one resource conflates the two
concerns.

**Why no per-target cost field on `TargetModel` in v1alpha1**

A natural follow-up question is whether `TargetModel.cost` should be added now
as an "override" so that in-cluster `modelServerName` targets can also
participate in cost-aware selection. v1alpha1 deliberately does not do this:

- External APIs price per token (USD / 1K tokens) — a clear, comparable unit.
- In-cluster `ModelServer` "cost" is GPU time × instance price ÷ utilization,
  amortized over many requests — a fundamentally different unit.
- Comparing the two requires a normalization model (e.g. converting GPU
  amortized cost into a synthetic per-token price) that is itself a
  non-trivial design problem.

Forcing users to fill a `TargetModel.cost` field before that normalization
model exists would produce inconsistent, unreliable inputs to `LeastCost`. The
follow-up proposal will resolve normalization first, then choose where to
attach per-target cost.

**Net effect for the reviewer's question**: the API can grow into cost-aware
routing without breaking existing users — `selectionPolicy` is the entry point,
and the data-source decision is intentionally postponed rather than locked in
prematurely.

#### 2. Customizable Secret Key

**Review**: "what `key` of the secret? Should we allow customize?"

**Response**: Changed `secretRef` from a string to a structured reference:

```yaml
auth:
  secretRef:
    name: openai-api-key
    key: apiKey          # Optional, defaults to "apiKey"
```

This follows standard Kubernetes patterns (e.g., `envFrom.secretKeyRef`) and allows users to reuse existing Secrets without creating new ones.

#### 3. Bearer Token as Default

**Review**: "Most maas requires bearer token"

**Response**: Simplified `auth` to use an enum with `Bearer` as the default:

```yaml
auth:
  type: Bearer           # Default; other options: APIKeyHeader, RawToken
  secretRef: { name, key }
```

**Behavior**:
- `type: Bearer` (default) → `Authorization: Bearer <token>`
- `type: APIKeyHeader` → `<headerName>: <token>` (e.g., `api-key: <token>` for Azure)
- `type: RawToken` → `Authorization: <token>` (no "Bearer " prefix)

90% of users get the right behavior with zero configuration.

#### 4. Relationship with ModelRoute

**Review**: "Explain how this works? We have model defined in modelRoute, so this need to match that one?"

**Response**: The new design eliminates this confusion by making `ExternalModelProvider` a **backend reference**, not an independent routing layer.

**How it works**:
1. `ModelRoute.spec.modelName` defines the client-facing model name
2. `ModelRoute.spec.rules[].targetModels[]` defines where traffic goes (in-cluster or external)
3. `TargetModel.externalProviderRef.modelName` specifies which upstream model to use

**Example**:
```yaml
kind: ModelRoute
spec:
  modelName: gpt-4-mini              # Client sees this
  rules:
    - name: default
      targetModels:
        - externalProviderRef:
            name: openai-gpt4
            modelName: gpt-4o-mini   # Provider receives this
          weight: 100
```

**No ambiguity**: There is no "matching" between `ModelRoute` and `ExternalModelProvider`. The relationship is explicit via `externalProviderRef`.

**Mixed routing**: Naturally supported by adding multiple `TargetModel` entries:
```yaml
rules:
  - name: default
    targetModels:
      - modelServerName: local-llama # 80% in-cluster
        weight: 80
      - externalProviderRef:         # 20% external
          name: openai-gpt4
          modelName: gpt-4o-mini
        weight: 20
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
  rules:
    - name: default
      targetModels:
        - externalProviderRef:
            name: openai
            modelName: gpt-4o-mini
          weight: 100
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
  rules:
    - name: default
      targetModels:
        - externalProviderRef:
            name: vllm-external
            modelName: meta-llama/Meta-Llama-3-70B-Instruct
          weight: 100
```

#### 3. Multi-Provider Load Balancing

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
  rules:
    - name: default
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

#### 4. Hybrid Routing (In-Cluster + External)

```yaml
apiVersion: networking.serving.volcano.sh/v1alpha1
kind: ModelRoute
metadata:
  name: gpt-4-mini-hybrid
  namespace: default
spec:
  modelName: gpt-4-mini
  rules:
    - name: default
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

### Extension Points

This proposal implements weight-based traffic splitting. The following extension points are designed into the API for future enhancements without breaking changes:

#### 1. Cost-Aware Routing

The API entry point already exists: `ModelRoute.spec.selectionPolicy` is a
struct, and a future proposal can widen its `type` enum to include `LeastCost`
without breaking changes. v1alpha1 admission rejects `LeastCost` outright
(rather than silently falling back) — see "Design Rationale §1" above for the
rationale, including why cost data is intentionally **not** modeled on
`ExternalModelProvider` in v1alpha1.

A follow-up proposal will pick the cost data source from the candidate list in
§1 and define request-time cost estimation. The field shape is already reserved;
turning the policy on only requires widening the `SelectionPolicy.Type` enum and
defining the new policy semantics.

#### 2. Latency-Aware Routing

A future proposal may widen `SelectionPolicy.Type` to include `LowestLatency`.
Observed latency data will be reported on `ExternalModelProvider.status` (e.g.
`status.observedLatency.p50Ms` / `p99Ms`) by the router's metrics pipeline.

```yaml
status:
  observedLatency:
    p50Ms: 120
    p99Ms: 350
```

When `selectionPolicy.type=LowestLatency` is supported in a future version,
router will select the target with the best latency profile. Like cost-aware
routing, this works uniformly for in-cluster and external targets.

#### 3. Quota and Rate-Limit Aware Routing

When an external provider returns HTTP 429 (rate limit exceeded), router can:
- Temporarily exclude it from selection
- Automatically failover to other providers serving the same model
- Emit metrics for alerting

#### 4. Regional Failover

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

### Alternatives Considered

#### 1. Independent Fallback Layer (Original Design)

**Rejected**: The original design had `ExternalModelProvider` as an independent routing layer with its own weight-based selection, separate from `ModelRoute`. This created:
- Two separate weight mechanisms (ModelRoute vs ExternalProvider)
- Confusion about the relationship between ModelRoute and ExternalProvider
- No path for mixed routing (in-cluster + external)
- Duplicate selection logic for future cost-aware routing

The new design unifies all routing through `ModelRoute`, making `ExternalModelProvider` a backend reference like `ModelServer`.

#### 2. Extend HTTPRoute with Auth

**Rejected**: HTTPRoute is generic and should not have LLM-specific features like model name rewriting and streaming response handling.

#### 3. Use Gateway API ExternalName Service

**Rejected**: Does not support Secret-based auth injection or model name rewriting.

#### 4. Deploy Separate Proxy Service

**Rejected**: Adds operational overhead and duplicates router functionality.

#### 5. Extend Gateway API `HTTPRoute.backendRefs` / Inference Extension

**Rejected for v1alpha1, kept under review for follow-up alignment**:

The Gateway API Inference Extension (kgateway / sig-network) is the natural
long-term home for in-cluster inference routing, and a maintainer may
reasonably ask why this proposal does not extend `HTTPRoute.backendRefs` to
point at a Secret-augmented external endpoint instead of introducing a new CRD.

Reasons for a kthena-native CRD in v1alpha1:

- **Co-evolution with `ModelRoute`**: `ExternalModelProvider` is consumed by
  `ModelRoute.rules[].targetModels[]` (a kthena-native target list), not by a generic Gateway
  resource. Forcing the relationship through `HTTPRoute.backendRefs` would
  require either a parallel `ModelRoute → HTTPRoute → backendRef` indirection
  or a custom `BackendObjectReference` group/kind — both of which are larger
  changes than this proposal.
- **Inference Extension scope**: the Gateway API Inference Extension
  (`InferencePool`) reached v1.0.0 GA in September 2025, but its scope is
  strictly in-cluster pod selection — `InferencePool` uses a pod label
  `selector` and `targetPorts`; it defines no external backend or
  Secret-authenticated upstream model. There is no upstream pattern to align
  with for this use case today.
- **Auth model gap**: Gateway API has no first-class concept for injecting
  Secret-backed auth headers per-backend; today, this requires policy
  attachment or a side-channel CRD. The shape of that abstraction in upstream
  is unresolved.

This is an explicit deferral, not a rejection. Once the Inference Extension
stabilizes a backend-auth model, kthena will evaluate migrating
`ExternalModelProvider` to a Gateway-aligned representation (or layering it on
top). The current CRD shape — narrow scope, no novel auth primitives — is
designed to make such migration straightforward.

### Questions

1. Should we support non-OpenAI-compatible APIs in a future iteration? (e.g., Anthropic Claude API, Google Gemini API)
2. Should we support Azure OpenAI's deployment-specific URL patterns and query parameters in the initial delivery or defer to a future iteration?
