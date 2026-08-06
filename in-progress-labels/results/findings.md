# Findings — issue #208

Run date: 2026-08-06. Cluster: docker-desktop, Kubernetes v1.34.1.

| Component | Version tested | Issue reported with |
|---|---|---|
| Argo CD | v3.5.0 | v3.3.6 |
| Argo Rollouts | v1.9.1 (chart 2.41.1) | v1.9.0 |
| Gateway API plugin | v0.16.0 | v0.13.0 |
| Data plane | Envoy Gateway v1.7.2 | Envoy Gateway v1.7.1 |

## Verdict

**Partially reproduced.** The diff asymmetry is real and demonstrated under controlled conditions. The
*"permanent, even at rest, with no rollout in progress"* framing is not — the HTTPRoute returns to `Synced` as soon as
the canary completes.

But the run turned up something worse than the cosmetic `OutOfSync` the issue body describes: **a sync during a canary
reverts the canary weights and silently sends 0% of traffic to the canary, with `ServerSideApply=true` *and*
`RespectIgnoreDifferences=true` set.** The issue's follow-up comment reports that only for the client-side /
`RespectIgnoreDifferences`-unset configuration.

| Claim | Result |
|---|---|
| Per-object jq asymmetry between live and desired | **Confirmed** |
| `OutOfSync` while the `in-progress` label is present | **Confirmed** |
| `OutOfSync` even when live `.spec.rules` is byte-identical to git | **Confirmed** |
| `ServerSideApply` + `RespectIgnoreDifferences` do not prevent it | **Confirmed** |
| Permanent / persists at rest with no rollout in progress | **Not reproduced** |
| Weights preserved, breakage is cosmetic only | **Refuted — weights are reverted on sync** |

## 1. The asymmetry (confirmed)

Argo CD evaluates `path(<expr>)` per object. Counting the paths it will strip:

```
live:    label=in-progress  -> paths stripped = 1
desired: label=<absent>     -> paths stripped = 0
```

So `.spec.rules` is removed from the live object and kept on the desired one. The rendered diff shows this as the
whole rules block being *added*, not as a weight change:

```
===== gateway.networking.k8s.io/HTTPRoute gatewayapi-demo/argo-rollouts-http-route ======
54a55,70
>   rules:
>   - backendRefs:
>     - group: ""
>       kind: Service
>       name: argo-rollouts-stable-service
>       port: 80
>       weight: 100
...
```

## 2. Controlled proof that the label alone causes it

While parked at `setWeight: 50`, the live `.spec.rules` were patched by hand back to git's exact values, so live and
desired were byte-identical:

| live `.spec.rules` | label | HTTPRoute status |
|---|---|---|
| identical to git | present | **OutOfSync** |
| identical to git | removed | **Synced** |

Nothing but the label changed between those two rows. This isolates the cause completely.

## 3. Weight reversion — the functional break

Config in force: `ServerSideApply=true`, `RespectIgnoreDifferences=true`, `selfHeal: false`.

While the canary was parked at `setWeight: 50`, a single `argocd app sync --resource ...HTTPRoute...`:

```
14:42:41 httproute=OutOfSync weights=[50 50] label=in-progress
--- argocd app sync ---
14:42:58 httproute=OutOfSync weights=[100 0]  label=in-progress
14:44:47 httproute=OutOfSync weights=[100 0]  label=in-progress     <- never restored
```

State afterwards:

- Rollout status: `canaryWeight=50 stableWeight=50`
- HTTPRoute: `stable=100, canary=0`
- Actual traffic: **40/40 requests to stable, 0 to the canary**
- `managedFields`: `argocd-controller (Apply)` owns `.spec.rules`; the `gatewayAPI` manager lost it

`RespectIgnoreDifferences` does not help, and for the same structural reason as the `OutOfSync`: the ignore is
evaluated against the *desired* object, which has no label, so no paths are protected and Argo CD applies git's
`.spec.rules` wholesale.

Two aggravating factors:

- **The plugin does not restore the weights.** It has no watch on the HTTPRoute and only writes during a Rollout
  reconcile, so the route stayed at `100/0` indefinitely. Verified separately: a hand-edit of the route while the
  rollout was paused survived 90 s untouched.
- **`selfHeal` was off.** Any sync will do it — a manual sync, or an unrelated commit landing while a canary is
  parked. With `selfHeal: true` it needs no human at all.

## 4. What is *not* reproduced

With the Service-selector noise also ignored (see section 5), the whole Application is `Synced` at rest, so the label
is the only thing that can move it. Re-running the canary on that clean baseline:

| State | App | HTTPRoute | Services | label |
|---|---|---|---|---|
| at rest, before | `Synced` | `Synced` | `Synced` | absent |
| parked at `setWeight: 50` | `OutOfSync` | **`OutOfSync`** | `Synced` | present |
| after full promotion, 1 active RS | `Synced` | **`Synced`** | `Synced` | absent |

After full promotion — single active ReplicaSet, one version, label removed, weights back to `100/0` — the HTTPRoute
returns to **`Synced`** and stays there.

The `OutOfSync` is scoped exactly to the window where the label is present. It is not permanent and it does not
survive the canary.

A paused canary does not count as "at rest": there are still two ReplicaSets and split traffic, so the rollout is by
definition in progress. The one state that genuinely *looks* finished while still carrying the label is a rollout
parked on a final `setWeight: 100` + `pause: {}` — confirmed `OutOfSync` with the label present at `Step 3/4` — which
is a plausible source of the reporter's "at rest" wording, but it is still an unfinished rollout.

## 5. Unrelated gotchas found on the way

Neither has anything to do with #208, but both produce `OutOfSync` and cost time if you assume they are the bug.

1. **Empty weights.** If `manifests/httproute.yaml` omits `backendRefs[].weight`, `backendRefs[].group` or
   `parentRefs[].group`/`kind`, the app is permanently `OutOfSync` at rest before any canary: the API server defaults
   `weight` to `1` and fills in group/kind, Argo CD is not doing a server-side diff, and the defaulted fields count as
   a difference. `argocd app diff` renders **nothing** in this state. It reproduces with `ignoreDifferences` removed
   entirely — that is how to tell it apart. Fixed here by pinning every defaulted field in git.
2. **Service selector hashes.** The Rollouts controller injects `rollouts-pod-template-hash` into
   `.spec.selector` of both the stable and canary Services, so both go `OutOfSync` after the first canary. A
   well-known Argo Rollouts + Argo CD wrinkle. Handled here by a second `ignoreDifferences` entry in
   `argocd/application.yaml`:

   ```yaml
   - group: ""
     kind: Service
     jsonPointers:
       - /spec/selector/rollouts-pod-template-hash
   ```

   Worth contrasting with the #208 expression: this one is **unconditional**, so it evaluates identically against the
   live and the desired object and causes no asymmetry. That is precisely the property the label-based snippet lacks.

Also worth noting: `argocd app sync --force` is rejected outright when `ServerSideApply=true`
(`error validating options: --force cannot be used with --server-side`).

## 6. Two candidate fixes, both tested

The docs' snippet cannot be made symmetric, because it keys on a label that exists only on the live object. Two
symmetric alternatives were tested in this harness. **The weight-scoped one wins.**

### 6a. `managedFieldsManagers` — fixes the symptoms, but ignores too much

```yaml
- group: gateway.networking.k8s.io
  kind: HTTPRoute
  managedFieldsManagers: [gatewayAPI]
```

It does fix both symptoms: `Synced` while parked at `setWeight: 50`, and a mid-canary `argocd app sync` left the
weights at `50/50` (traffic 18 blue / 22 green) instead of starving the canary.

But it ignores far more than intended. The plugin mutates the route with a full-object `Update()`, so it claims the
whole subtree atomically:

```json
{"manager": "gatewayAPI", "operation": "Update", "fieldsV1": {"f:spec": {"f:rules": {}}}}
```

`f:rules: {}` with no nested keys = the entire `.spec.rules` list. Verified consequence — live path changed to
`/LIVE-CHANGED` while git says `/`, patched as `--field-manager=gatewayAPI` so ownership was retained:

| ignore config | live `/LIVE-CHANGED` vs git `/` |
|---|---|
| label snippet | `OutOfSync` (detected) |
| `managedFieldsManagers: [gatewayAPI]` | **`Synced` (invisible)** |

And this applies **at rest**, not just during a canary, because the plugin writes weights on the final promotion and
keeps ownership afterwards. So real drift or real git changes to the routing rules would be silently unreconciled
forever. That is *worse* than the label form, which at least reconciles rules between canaries. Not recommended.

### 6b. Weight-scoped, unconditional — recommended

```yaml
- group: gateway.networking.k8s.io
  kind: HTTPRoute
  jqPathExpressions:
    - .spec.rules[].backendRefs[].weight
```

Symmetric by construction: the path exists on the live *and* the desired object (git pins the weights), so
`select()`-style asymmetry cannot arise. Measured:

| | label snippet | `managedFieldsManagers` | **weight-scoped** |
|---|---|---|---|
| at rest | `Synced` | `Synced` | `Synced` |
| parked at `setWeight: 50` | **`OutOfSync`** | `Synced` | **`Synced`** |
| weights after mid-canary sync | **`100/0` starved** | `50/50` | **`50/50`** |
| traffic after that sync | 40/40 stable | 18/22 | **20/20** |
| structural rules change detected | yes | **no** | **yes** |

It gives up GitOps reconciliation of the weight *values* — which is exactly the intent, since the plugin owns them —
while leaving everything else about the rules under Argo CD's control. No label, no static git annotation, no
hand-maintained rule names.

For header-based routing the plugin injects whole rules, so that case additionally needs the rule-name select already
documented in `docs/features/header-based-routing.md`. Not tested here (out of scope).

## 7. Suggested follow-up for the issue

Given finding 3, the docs need more than a caveat: the recommended snippet does not merely produce a cosmetic
`OutOfSync`, it leaves the canary reachable by 0% of traffic if anything syncs the app mid-rollout.

1. Replace the label snippet in `docs/quick-start.md:273-291` and `docs/features/multiple-routes.md:113-137` with the
   weight-scoped form from 6b, and note that the old one can revert weights. Do **not** recommend
   `managedFieldsManagers` — see 6a.
2. Leave the label itself in place — it is a useful marker for alerting and `kubectl get -l` — but stop presenting it
   as the Argo CD integration, since that is its only documented purpose today. Users who want it gone already have
   `disableInProgressLabel: true`.
3. The reporter's static-git-label workaround is sound but strictly more work: it needs a label added to git and
   rule names kept in sync by hand.
