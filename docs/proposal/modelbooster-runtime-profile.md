---
title: Runtime Profile CRD for ModelBooster
authors:
- "@xrwang8"
reviewers:
- TBD
approvers:
- TBD

creation-date: 2026-05-06

---

## Runtime Profile CRD for ModelBooster

### Summary

Add a declarative runtime profile API for `ModelBooster`, similar in spirit to KServe's `ServingRuntime` / `ClusterServingRuntime`, so cluster administrators can define reusable model runtime templates outside the controller binary.

Today `ModelBooster` is the right user-facing abstraction for productized model deployment, but the runtime conversion is still mostly controller-owned. For example, vLLM and vLLM disaggregated deployments are selected from embedded templates in `pkg/model-booster-controller/convert/templates/`, and adding a new runtime shape such as SGLang single-node or SGLang PD disaggregation requires code changes.

### Motivation

For product/platform users, `ModelBooster` should hide low-level `ModelServing` details such as roles, entry/worker pod templates, runtime sidecars, PD topology, model downloader wiring, metrics endpoints, and runtime-specific commands.

However, platform operators still need a supported way to define and maintain runtime presets. The current embedded template approach has several long-term limitations:

**1. New runtimes require controller code changes**

Today the runtime selection is effectively compiled into the controller:

- `vLLM` → `pkg/model-booster-controller/convert/templates/vllm.yaml`
- `vLLMDisaggregated` → `pkg/model-booster-controller/convert/templates/vllm-pd.yaml`

Adding SGLang single-node, SGLang PD disaggregation, MindIE, private vLLM profiles, or hardware-specific profiles requires changing controller code, rebuilding the image, and rolling out the controller.

**2. Templates are not Kubernetes resources**

The templates are embedded with Go `embed`. Cluster administrators cannot add, patch, validate, or version runtime templates through normal Kubernetes workflows such as `kubectl apply`, GitOps, RBAC, admission policy, or status conditions.

**3. Cluster-specific runtime differences are hard to operate**

Real deployments often need different defaults per cluster or tenant (private/offline engine images, GPU vs NPU resource names, preinstalled NIXL/Mooncake dependencies, `nodeSelector`, affinity, tolerations, `/dev/shm` sizing, probes and startup timeouts, security context and image pull secrets, runtime metrics ports and paths).

**4. Product UX is forced to hardcode profiles elsewhere**

A product console usually wants to expose simple choices such as `vllm-single`, `vllm-pd`, `sglang-single`, `sglang-pd`. Without a first-class runtime profile API, these choices must be hardcoded in the console, CLI, or controller logic.

**5. Small runtime changes require a controller release**

Adjusting a probe, command, image, env var, resource default, or volume currently means modifying embedded templates and shipping a new controller image.

**6. No runtime auto-selection or priority model**

KServe's `ClusterServingRuntime` can declare supported formats with `autoSelect` and `priority`. Kthena does not currently have an equivalent mechanism.

**7. Community contributions are heavier than necessary**

Contributing a new runtime shape currently requires touching controller conversion code and tests.

This proposal makes `ModelBooster` a cleaner northbound/product API while leaving `ModelServing` as the generated low-level workload API.

#### Goals

1. Add a cluster-scoped `ClusterModelRuntimeProfile` CRD.
2. Support explicit `spec.backend.runtimeProfile` on `ModelBooster`.
3. Move the existing embedded `vLLM` and `vLLMDisaggregated` templates into default installed profiles (`vllm-single`, `vllm-pd`).
4. When `runtimeProfile` is unset, fall back to current embedded template behavior — no behavior change for existing users.
5. Add validation to ensure the selected profile supports the requested backend type.
6. Document how operators can add profiles for SGLang and private engine images.

#### Non-Goals

- Namespace-scoped `ModelRuntimeProfile` (deferred to later phase)
- Auto-selection with `autoSelect` / `priority` (deferred to later phase)
- Status conditions for invalid or missing profiles (deferred to later phase)
- Profile versioning (deferred to later phase)
- SGLang profiles (can be contributed as profile YAMLs once the CRD exists)
- PD/multinode structured role templates replacing current placeholder approach (deferred to later phase)

### Proposal

Introduce a new `ClusterModelRuntimeProfile` CRD. The profile mirrors the actual structure of the current embedded templates: each role has an `entryTemplate` (the pod that runs both the kthena `runtime` sidecar and the inference `engine` container) and an optional `workerTemplate` (for multi-node ray workers).

`ModelBooster` can reference a profile explicitly:

```yaml
spec:
  backend:
    runtimeProfile: vllm-single
    modelURI: hf://Qwen/Qwen2.5-7B-Instruct
    minReplicas: 1
    maxReplicas: 4
```

When `runtimeProfile` is unset, the controller falls back to the current embedded template behavior, preserving full backward compatibility.

#### KServe Reference

KServe has a useful pattern here:

- `ServingRuntime` / `ClusterServingRuntime` are Kubernetes resources used as runtime templates.
- A runtime declares supported model formats through `supportedModelFormats` with fields such as `name`, `version`, `autoSelect`, and `priority`.
- A runtime carries pod-level details such as containers, args, env, resources, probes, volumes, affinity, annotations, and protocol versions.
- The KServe multinode HuggingFace runtime includes a `workerSpec`, which is close to Kthena's need to generate entry/worker or prefill/decode role topology.

References:

- KServe Serving Runtime docs: https://kserve.github.io/website/docs/concepts/resources/servingruntime
- KServe `ClusterServingRuntime` CRD: https://github.com/kserve/kserve/blob/master/charts/kserve-crd/templates/serving.kserve.io_clusterservingruntimes.yaml
- KServe HuggingFace runtime example: https://github.com/kserve/kserve/blob/master/config/runtimes/kserve-huggingfaceserver.yaml
- KServe multinode HuggingFace runtime example: https://github.com/kserve/kserve/blob/master/config/runtimes/kserve-huggingfaceserver-multinode.yaml

### Design Details

#### CRD Structure

The `ClusterModelRuntimeProfile` CRD has the following structure:

```yaml
apiVersion: workload.serving.volcano.sh/v1alpha1
kind: ClusterModelRuntimeProfile
metadata:
  name: vllm-single
spec:
  disabled: false
  supportedBackends:
    - type: vLLM
      autoSelect: true
      priority: 10
  protocolVersions:
    - openai-v1
  defaults:
    runtimeImage: ghcr.io/volcano-sh/kthena-runtime:v0.10.0  # kthena sidecar
    engineImage: vllm/vllm-openai:v0.8.5                     # inference engine
    metricsPath: /metrics
    metricsPort: 8000
    runtimePort: 8100
  modelServingTemplate:
    roles:
      - name: leader
        entryTemplate:
          spec:
            terminationGracePeriodSeconds: 300
            containers:
              - name: runtime          # kthena sidecar — proxies requests, exposes /health
                image: "{{ .defaults.runtimeImage }}"
                ports:
                  - containerPort: "{{ .defaults.runtimePort }}"
                args:
                  - --port
                  - "{{ .defaults.runtimePort }}"
                  - --engine
                  - "{{ .backendType }}"
                  - --engine-base-url
                  - "http://localhost:{{ .defaults.metricsPort }}"
                  - --engine-metrics-path
                  - "{{ .defaults.metricsPath }}"
                  - --pod
                  - "$(POD_NAME).$(NAMESPACE)"
                  - --model
                  - "{{ .modelName }}"
                readinessProbe:
                  httpGet:
                    path: /health
                    port: "{{ .defaults.runtimePort }}"
                  initialDelaySeconds: 5
                  periodSeconds: 10
              - name: engine           # inference engine — runs the actual model
                image: "{{ .workerImage }}"
                command: "{{ .engineCommand }}"
                resources: "{{ .workerResources }}"
                volumeMounts: "{{ .volumeMounts }}"
                readinessProbe:
                  httpGet:
                    path: /health
                    port: "{{ .defaults.metricsPort }}"
                  initialDelaySeconds: 180
                  periodSeconds: 5
                  failureThreshold: 3
            initContainers: "{{ .initContainers }}"
            volumes: "{{ .volumes }}"
        workerTemplate:               # ray worker pods for multi-node
          spec:
            containers:
              - name: "{{ .backendName }}-{{ .backendType }}-worker"
                image: "{{ .workerImage }}"
                command:
                  - bash
                  - "-c"
                  - "chmod u+x examples/online_serving/multi-node-serving.sh && examples/online_serving/multi-node-serving.sh worker --ray_address=$(ENTRY_ADDRESS)"
                resources: "{{ .workerResources }}"
                volumeMounts: "{{ .volumeMounts }}"
            initContainers: "{{ .initContainers }}"
            volumes: "{{ .volumes }}"
```

#### vLLM PD Profile

The PD profile has two independent roles — `prefill` and `decode` — each with its own engine image, env, and command:

```yaml
apiVersion: workload.serving.volcano.sh/v1alpha1
kind: ClusterModelRuntimeProfile
metadata:
  name: vllm-pd
spec:
  disabled: false
  supportedBackends:
    - type: vLLMDisaggregated
      autoSelect: true
      priority: 10
  protocolVersions:
    - openai-v1
  defaults:
    runtimeImage: ghcr.io/volcano-sh/kthena-runtime:v0.10.0
    metricsPath: /metrics
    metricsPort: 8000
    runtimePort: 8100
  modelServingTemplate:
    roles:
      - name: prefill
        entryTemplate:
          spec:
            containers:
              - name: runtime
                image: "{{ .defaults.runtimeImage }}"
                ports:
                  - containerPort: "{{ .defaults.runtimePort }}"
                args:
                  - --port
                  - "{{ .defaults.runtimePort }}"
                  - --engine
                  - "{{ .backendType }}"
                  - --engine-base-url
                  - "http://localhost:{{ .defaults.metricsPort }}"
                  - --engine-metrics-path
                  - "{{ .defaults.metricsPath }}"
                  - --pod
                  - "$(POD_NAME).$(NAMESPACE)"
                  - --model
                  - "{{ .modelName }}"
              - name: vllm
                image: "{{ .prefillWorkerImage }}"
                command: "{{ .prefillCommand }}"
                resources: "{{ .prefillResources }}"
                volumeMounts: "{{ .volumeMounts }}"
                readinessProbe:
                  httpGet:
                    path: /health
                    port: "{{ .defaults.metricsPort }}"
                  initialDelaySeconds: 5
                  periodSeconds: 5
                  failureThreshold: 3
                livenessProbe:
                  httpGet:
                    path: /health
                    port: "{{ .defaults.metricsPort }}"
                  initialDelaySeconds: 180
                  periodSeconds: 5
                  failureThreshold: 3
            initContainers: "{{ .initContainers }}"
            volumes: "{{ .volumes }}"
        workerReplicas: 0
      - name: decode
        entryTemplate:
          spec:
            containers:
              - name: runtime
                image: "{{ .defaults.runtimeImage }}"
                ports:
                  - containerPort: "{{ .defaults.runtimePort }}"
                args:
                  - --port
                  - "{{ .defaults.runtimePort }}"
                  - --engine
                  - "{{ .backendType }}"
                  - --engine-base-url
                  - "http://localhost:{{ .defaults.metricsPort }}"
                  - --engine-metrics-path
                  - "{{ .defaults.metricsPath }}"
                  - --pod
                  - "$(POD_NAME).$(NAMESPACE)"
                  - --model
                  - "{{ .modelName }}"
              - name: vllm
                image: "{{ .decodeWorkerImage }}"
                command: "{{ .decodeCommand }}"
                resources: "{{ .decodeResources }}"
                volumeMounts: "{{ .volumeMounts }}"
                readinessProbe:
                  httpGet:
                    path: /health
                    port: "{{ .defaults.metricsPort }}"
                  initialDelaySeconds: 5
                  periodSeconds: 5
                  failureThreshold: 3
                livenessProbe:
                  httpGet:
                    path: /health
                    port: "{{ .defaults.metricsPort }}"
                  initialDelaySeconds: 180
                  periodSeconds: 5
                  failureThreshold: 3
            initContainers: "{{ .initContainers }}"
            volumes: "{{ .volumes }}"
        workerReplicas: 0
```

#### Template Placeholder Format

The sketches above use Go template syntax (`{{ .field }}`). The current embedded templates use `${VAR}` string substitution. For phase 1, we lean toward Go templates since they support conditionals and range, which will be needed for optional fields.

#### Backward Compatibility

When `runtimeProfile` is unset on `ModelBooster`, the controller falls back to the current embedded template behavior. This ensures no behavior change for existing users.

The existing embedded templates will be migrated to default installed profiles (`vllm-single`, `vllm-pd`) that ship with the controller, but the fallback logic ensures smooth migration.

#### Risks and Mitigations

**Risk**: Template placeholder injection vulnerabilities if user-provided values are not properly escaped.

**Mitigation**: Use Go's `text/template` with proper escaping. Validate all placeholder values before substitution.

**Risk**: Profile versioning and compatibility across controller upgrades.

**Mitigation**: Phase 1 does not include versioning. Future phases can add `apiVersion` or `schemaVersion` fields to profiles.

### Test Plan

1. **Unit tests**: Test profile loading, validation, and template rendering with various placeholder values.
2. **Integration tests**: Deploy `ModelBooster` with explicit `runtimeProfile` and verify generated `ModelServing` matches expected structure.
3. **E2E tests**: Deploy vllm-single and vllm-pd profiles, create `ModelBooster` instances, verify pods start correctly and serve requests.
4. **Backward compatibility tests**: Deploy `ModelBooster` without `runtimeProfile` and verify fallback to embedded templates works.
5. **Validation tests**: Attempt to reference non-existent profiles, mismatched backend types, and verify validation webhooks reject invalid configurations.

### Alternatives

#### Alternative 1: Keep embedded templates, add overlay mechanism

Instead of full CRD-based profiles, add a `ModelBooster` field for template overlays (e.g., `spec.backend.templateOverrides`). This would allow per-instance customization without requiring cluster-scoped resources.

**Rejected because**: This does not solve the problem of cluster operators curating a catalog of runtime offerings. It also makes `ModelBooster` more complex.

#### Alternative 2: Use ConfigMaps for templates

Store templates in ConfigMaps instead of a CRD. The controller watches ConfigMaps and loads templates dynamically.

**Rejected because**: ConfigMaps lack schema validation, versioning, and RBAC granularity. A CRD provides better type safety and operational semantics.

#### Alternative 3: CLI manifest templates remain independent

The CLI currently generates `ModelServing` manifests directly. One option is to keep CLI templates independent and only use profiles for `ModelBooster` controller-generated resources.

**Decision deferred**: Phase 1 focuses on controller-generated resources. CLI integration can be addressed in a later phase.
