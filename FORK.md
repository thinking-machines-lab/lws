# Fork: controller-driven rolling updates (`rollout-via-delete`)

This fork of [kubernetes-sigs/lws](https://github.com/kubernetes-sigs/lws) adds one
opt-in feature. Everything else tracks upstream.

## Why

Upstream LWS paces rolling updates through the leader statefulset's
`rollingUpdate.maxUnavailable`, which requires the `MaxUnavailableStatefulSet` feature
gate (alpha, unavailable on managed clusters such as GKE). Without the gate, the
statefulset controller recreates leaders strictly one at a time, gated on readiness.
For multi-node inference groups where leader-ready means model-loaded, a rollout of N
groups takes N cold starts in sequence.

## Usage

Annotate a LeaderWorkerSet:

```yaml
metadata:
  annotations:
    leaderworkerset.sigs.k8s.io/rollout-via-delete: "true"
```

With the annotation, the leader statefulset uses the `OnDelete` update strategy and the
lws controller deletes stale leader pods itself, in parallel, at the concurrency allowed
by the group-level `maxUnavailable`/`maxSurge` budget. The partition arithmetic is
unchanged from upstream; the partition is tracked in the
`leaderworkerset.sigs.k8s.io/update-partition` annotation on the leader statefulset
(`OnDelete` forbids `spec.updateStrategy.rollingUpdate`), and availability is re-checked
at deletion time so concurrent failures pause the rollout instead of stacking on top of
it.

Without the annotation (the default), behavior is upstream: statefulset-driven updates
via the partition field. Deploying this controller build is therefore a no-op until
objects are annotated, and the feature can be enabled per object (per namespace, per
pool).

## Rollback

Set the annotation to `"false"` (or remove it). The controller restores the
`RollingUpdate` strategy on the leader statefulset, carrying the current partition over,
without disturbing pods or revisions. This is safe mid-rollout.

## Downgrading the controller binary

**Disable `rollout-via-delete` on every LeaderWorkerSet first**, and wait for all leader
statefulsets to show `updateStrategy.type: RollingUpdate` again:

```bash
kubectl get sts -A -l leaderworkerset.sigs.k8s.io/name \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,STRATEGY:.spec.updateStrategy.type
```

Upstream controller builds dereference the leader statefulset's `rollingUpdate` config
unconditionally and will panic-loop on the `OnDelete` statefulsets this mode creates. If
a downgrade already happened, recover per statefulset with:

```bash
kubectl patch sts <lws-name> -n <ns> --type=merge -p \
  '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":0}}}}'
```

## Other divergences from upstream

- Percentage budgets resolve like Deployments: only literal `0`/`0` is rejected at
  admission, and a resolved `maxUnavailable` of zero is floored at 1, so
  `maxUnavailable: 20%` cannot stall a rollout at small replica counts.
- Availability accounting treats terminating leaders/worker statefulsets as unavailable
  and only trusts pods and worker statefulsets that are controller-owned by the
  expected parent.
