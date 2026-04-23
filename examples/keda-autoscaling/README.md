# KEDA Autoscaling for ModelServing

This example autoscales a `ModelServing` using [KEDA](https://keda.sh/) driven
by a Prometheus query against kthena-router metrics.

## Prerequisites

- KEDA installed in the `keda` namespace.
- A Prometheus stack reachable at the `serverAddress` used in `scaledobject.yaml`
  (the example assumes kube-prometheus-stack in the `monitoring` namespace).
- kthena-router deployed and serving traffic for a model known to Prometheus
  under the `model` label (see "Model label" below).

## Files

- `modelserving.yaml`   — a minimal ModelServing named `test-model`.
- `servicemonitor.yaml` — scrapes kthena-router `/metrics` into Prometheus.
- `rbac.yaml`           — grants the KEDA operator access to the ModelServing
  `scale` subresource (required for KEDA to change `spec.replicas`).
- `scaledobject.yaml`   — the KEDA `ScaledObject` that drives scaling.

## Usage

```bash
kubectl apply -f rbac.yaml
kubectl apply -f servicemonitor.yaml
kubectl apply -f modelserving.yaml
kubectl apply -f scaledobject.yaml
```

## Model label

The Prometheus query filters by `model="test"`:

```promql
sum(kthena_router_active_downstream_requests{model="test"})
```

The `model` label is populated by kthena-router from the request (e.g. the
`model` field in an OpenAI-style payload, or the ModelRoute match). Change the
label value to match the model name your traffic uses — a global `sum(...)`
without the filter would aggregate every model on the router and produce wrong
scaling decisions.

## Scaling unit: ServingGroup, not pod

`spec.replicas` on a ModelServing is the number of **ServingGroups**, and each
group can contain several pods (one entry + workers per role). The scale
subresource's `status.labelSelector` is therefore scoped to the entry pod of
the first role, so the selector matches exactly one pod per group and the
HPA/KEDA pod count lines up with `spec.replicas`.
