# cloudnative-pg

CloudNativePG operator manifests, deployed to multiple clusters via Kustomize overlays.

## Structure

- `base/` — rendered Helm template output for the operator + Barman Cloud Plugin manifest. Cluster-agnostic.
- `overlays/<cluster>/` — per-cluster overlay. References `../../base` and adds patches if needed. ArgoCD Applications point at an overlay, never at `base/` directly.

## Upgrades

When upgrading the operator or the Barman Cloud Plugin, re-render the manifest into `base/` (see README for exact `helm template` / `curl` commands), commit on a feature branch, open a PR. After merge, bump the `targetRevision` on each affected Application in `internal-applications-kub`.

## When adding a new cluster

1. Create `overlays/<cluster>/kustomization.yaml` referencing `../../base`. Start with no patches.
2. If the cluster needs cluster-specific config (StorageClass, pull secrets, etc.), add it as a patch in that overlay.
3. Open a PR in `internal-applications-kub` adding the new ArgoCD Application + AppProject. AppProject name must equal the destination namespace per iak's policy.

## Verifying a render

```bash
kubectl kustomize overlays/oke
```

The output should match what's in `base/` plus any overlay patches. If it errors on a positional patch, the upstream chart probably bumped — re-pin the patch before merging.
