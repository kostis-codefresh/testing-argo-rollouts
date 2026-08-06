# Reproducing gatewayapi-plugin issue #208

Harness for [argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi#208](https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/issues/208).

## The claim

The plugin's [`docs/quick-start.md`](https://argo-rollouts-gateway-api.readthedocs.io/en/latest/quick-start/)
recommends this `ignoreDifferences` expression so Argo CD tolerates the route edits a canary makes:

```
select(.metadata.labels["rollouts.argoproj.io/gatewayapi-canary"] == "in-progress") | .spec.rules
```

Issue #208 reports that on Argo CD 3.x this produces a **permanent, unreconcilable `OutOfSync`**, and that it
persists even with `ServerSideApply=true` + `RespectIgnoreDifferences=true`.

The proposed mechanism is an asymmetry. Argo CD evaluates `jqPathExpressions` per object, against the live object
*and* the desired (git) object. The plugin stamps `rollouts.argoproj.io/gatewayapi-canary=in-progress` at runtime, so
the label only ever exists on the live object:

| | label present? | `select(...)` matches? | `.spec.rules` masked? |
|---|---|---|---|
| live | yes | yes | yes |
| desired (git) | no | no | no |

So Argo CD ends up comparing a rules-less live object against a rules-present desired object.

Relevant plugin source (`rollouts-plugin-trafficrouter-gatewayapi`):

- `internal/defaults/defaults.go:3-6` — the label key/value constants. There is no `complete` value; "off" is
  absence of the key.
- `pkg/plugin/labels.go:9-42` — `ensureInProgressLabel` adds the label on any `SetWeight` with
  `desiredWeight != 0` and removes it only at `desiredWeight == 0`.
- `pkg/plugin/httproute.go:61-63` — called immediately before a full-object `Update()`.

One consequence worth noting up front: a rollout parked on a final `setWeight: 100` + `pause: {}` still carries the
label, because the plugin has not yet been called with weight 0.

## Scope

Scenario A only — the configuration in the issue body. The follow-up comment's variant (client-side apply +
`RespectIgnoreDifferences` unset, where the canary weight is actually reverted), the static-label workaround, and the
plugin's own `disableInProgressLabel` knob are not covered here.

## Layout

```
infra/       kubectl/helm-applied, outside the Argo CD Application
manifests/   the Argo CD source path — the only thing under test
argocd/      the Application, applied with kubectl
results/     evidence captured during a run
```

Only `manifests/` is synced by Argo CD, so the diff under test is exclusively the HTTPRoute.

## Versions

| Component | Version | Issue reported with |
|---|---|---|
| Argo CD | v3.5.0 | v3.3.6 |
| Argo Rollouts | v1.9.1 (chart 2.41.1) | v1.9.0 |
| Gateway API plugin | v0.16.0 | v0.13.0 |
| Data plane | Envoy Gateway v1.7.2 | Envoy Gateway v1.7.1 |

## Run it

### 1. Envoy Gateway

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.7.2 \
  -n envoy-gateway-system --create-namespace
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

### 2. Argo Rollouts + the Gateway API plugin

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-rollouts argo/argo-rollouts --version 2.41.1 \
  -n argo-rollouts --create-namespace -f infra/argo-rollouts-values.yaml --wait
```

The controller downloads the 78 MB plugin binary at startup, so check it actually got it:

```bash
kubectl -n argo-rollouts logs deploy/argo-rollouts | grep -i plugin
```

Chart 2.41.1 defaults `providerRBAC.providers.gatewayAPI: true`, which already grants `httproutes`
`get/list/watch/update` — no extra ClusterRole needed.

### 3. Namespace and Gateway

```bash
kubectl create ns gatewayapi-demo
kubectl apply -f infra/gateway.yaml
kubectl get gateway -n gatewayapi-demo eg     # PROGRAMMED should be True
```

### 4. The Argo CD Application

`manifests/` must be committed and pushed first — Argo CD reads it from git, not from disk.

```bash
kubectl apply -f argocd/application.yaml
argocd app get gatewayapi-in-progress-labels
```

`selfHeal` is deliberately **off**. The Rollout spec is owned by Argo CD, so with `selfHeal: true` a
`kubectl argo rollouts set image` is reverted within seconds and the canary never starts. It is not needed to
reproduce #208 — as section 3 of the findings shows, a single explicit `argocd app sync` is enough to cause the
weight reversion, `selfHeal` merely removes the need for a human to trigger it.

**Baseline check:** the app must be `Synced` / `Healthy` here, with the rollout at 5/5 and no
`gatewayapi-canary` label on the HTTPRoute. If it is already `OutOfSync` at rest, something else is wrong and
nothing below means anything.

> **Gotcha, unrelated to #208 — empty weights.** If `manifests/httproute.yaml` omits `backendRefs[].weight`
> (or `parentRefs[].group`/`kind`, or `backendRefs[].group`), the app is permanently `OutOfSync` at rest before any
> canary runs: the API server defaults `weight` to `1` and fills in the group/kind, Argo CD is not doing a
> server-side diff, and the defaulted fields register as a difference. Confusingly `argocd app diff` renders
> *nothing* in that state. It reproduces with `ignoreDifferences` removed entirely, which is how you can tell it
> apart from the real issue. The manifest here pins every defaulted field so the baseline is clean.
>
> **Second unrelated gotcha — service selector hashes.** The Rollouts controller injects
> `rollouts-pod-template-hash` into `.spec.selector` of the stable and canary Services, so both go `OutOfSync` after
> the first canary. `argocd/application.yaml` carries a second `ignoreDifferences` entry for it. Unlike the #208
> expression that one is unconditional, so it evaluates identically on live and desired — which is exactly the
> property the label-based snippet lacks.
>
> With both out of the way the whole Application is `Synced` at rest, so the `in-progress` label is the only thing
> that can move it.

### 5. Confirm the data plane

```bash
kubectl -n envoy-gateway-system port-forward \
  "svc/$(kubectl -n envoy-gateway-system get svc -o name | grep gatewayapi-demo-eg | cut -d/ -f2)" 8080:80 &
for i in (seq 20); curl -s -H 'Host: demo.example.com' localhost:8080/color; end
```

All responses should be `"blue"`.

### 6. Trigger the canary

```bash
kubectl argo rollouts set image rollouts-demo \
  rollouts-demo=argoproj/rollouts-demo:green -n gatewayapi-demo
kubectl argo rollouts get rollout rollouts-demo -n gatewayapi-demo --watch
```

It advances to `setWeight: 50` and pauses.

### 7. Measure at `setWeight: 50`

This is the actual test.

```bash
# the plugin's edits: label present, weights 50/50
kubectl get httproute argo-rollouts-http-route -n gatewayapi-demo -o yaml

# the question the issue asks
argocd app get gatewayapi-in-progress-labels
argocd app diff gatewayapi-in-progress-labels

# the per-object asymmetry, straight from the issue body
J=$(kubectl get httproute argo-rollouts-http-route -n gatewayapi-demo -o json)
set L 'rollouts.argoproj.io/gatewayapi-canary'
echo $J | jq --arg L "$L" '.metadata.labels[$L]="in-progress" | [select(.metadata.labels[$L]=="in-progress")|.spec.rules]|(.[0]|length)//0'   # live    => N
echo $J | jq --arg L "$L" '.metadata.labels[$L]=null          | [select(.metadata.labels[$L]=="in-progress")|.spec.rules]|(.[0]|length)//0'   # desired => 0

# do the weights hold, and who owns .spec.rules?
for i in (seq 12)
  kubectl get httproute argo-rollouts-http-route -n gatewayapi-demo -o json | \
    jq -c '{w: [.spec.rules[].backendRefs[].weight], owners: [.metadata.managedFields[] | select(.fieldsV1|tostring|contains("rules")) | .manager]}'
  sleep 5
end
```

### 7b. The controlled test — is it the label, or the weights?

Patch the live rules back to git's exact values while the label is still there, then remove the label. Nothing but
the label changes between the two readings:

```bash
kubectl -n gatewayapi-demo patch httproute argo-rollouts-http-route --type=json \
  -p '[{"op":"replace","path":"/spec/rules/0/backendRefs/0/weight","value":100},
       {"op":"replace","path":"/spec/rules/0/backendRefs/1/weight","value":0}]'
argocd app get gatewayapi-in-progress-labels --hard-refresh | grep HTTPRoute   # OutOfSync

kubectl -n gatewayapi-demo label httproute argo-rollouts-http-route \
  'rollouts.argoproj.io/gatewayapi-canary-'
argocd app get gatewayapi-in-progress-labels --hard-refresh | grep HTTPRoute   # Synced
```

### 7c. The weight reversion

Still parked at `setWeight: 50`, sync the HTTPRoute once and watch the canary lose all its traffic:

```bash
argocd app sync gatewayapi-in-progress-labels \
  --resource 'gateway.networking.k8s.io:HTTPRoute:argo-rollouts-http-route'

kubectl -n gatewayapi-demo get rollout rollouts-demo \
  -o jsonpath='{.status.canary.weights.canary.weight}{"\n"}'          # 50
kubectl -n gatewayapi-demo get httproute argo-rollouts-http-route \
  -o jsonpath='{.spec.rules[0].backendRefs[*].weight}{"\n"}'          # 100 0
for i in $(seq 40); do curl -s -H 'Host: demo.example.com' localhost:8080/color; echo; done | sort | uniq -c
kubectl -n gatewayapi-demo get httproute argo-rollouts-http-route --show-managed-fields -o json | \
  jq -r '.metadata.managedFields[] | "\(.manager)\t\(.operation)\trules=\((.fieldsV1|tostring)|contains("rules"))"'
```

The plugin will **not** put the weights back — it has no watch on the HTTPRoute and only writes during a Rollout
reconcile.

### 8. Promote and re-measure

```bash
kubectl argo rollouts promote rollouts-demo -n gatewayapi-demo   # -> setWeight: 100 + pause
argocd app get gatewayapi-in-progress-labels
kubectl get httproute argo-rollouts-http-route -n gatewayapi-demo -o jsonpath='{.metadata.labels}'

kubectl argo rollouts promote rollouts-demo -n gatewayapi-demo   # -> fully promoted, weight 0, label dropped
argocd app get gatewayapi-in-progress-labels
```

The second promote is what settles the issue's "permanent, even at rest" wording: if the app goes back to `Synced`
once the label is gone, the `OutOfSync` is scoped to the canary window rather than permanent.

Note that a *paused* canary is not "at rest" — there are still two ReplicaSets and traffic is split, so the rollout is
in progress by definition. The at-rest test is the fully-promoted state: one active ReplicaSet, one version.

```bash
kubectl -n gatewayapi-demo get rs -o json | jq -r '.items[] | select(.spec.replicas>0) | .metadata.name'
```

## Findings

See [`results/findings.md`](results/findings.md).

## Teardown

```bash
kubectl delete -f argocd/application.yaml
helm uninstall argo-rollouts -n argo-rollouts
helm uninstall eg -n envoy-gateway-system
kubectl delete ns gatewayapi-demo envoy-gateway-system argo-rollouts
kubectl delete gatewayclass eg
```
