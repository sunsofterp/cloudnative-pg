# cloudnative-pg

CloudNativePG operator manifests, deployed to multiple clusters via Kustomize overlays.

## Structure

- `vendir.yml` / `vendir.lock.yml` at the repo root pin external manifest artifacts (Barman Cloud Plugin). Run `vendir sync` from the repo root to regenerate `components/barman-cloud-plugin/vendor/`. The synced content is committed.
- `base/kustomization.yaml` renders the OPERATOR ONLY from the upstream Helm chart at build time via `helmCharts:` (chart `cloudnative-pg`, repo `https://cloudnative-pg.github.io/charts`). The chart's resource limits/requests come from `valuesInline.resources`.
- `components/barman-cloud-plugin/` is the Barman Cloud Plugin as an OPT-IN kustomize Component: the vendored manifest, the label/resource patches, and its own `namespace: cnpg-system` transform (components don't inherit the base's). Opt-in because the manifest carries cert-manager Certificate/Issuer resources — it can only land on clusters that run cert-manager.
- `base/charts/` is **gitignored** — kustomize pulls the chart fresh on each `--enable-helm` build, both locally and in ArgoCD's repo-server. Don't commit anything from there. (The `.gitignore` exists for this reason.)
- `overlays/<cluster>/` references `../../base`, plus `components: [../../components/barman-cloud-plugin]` where the plugin belongs. `overlays/oke/` and `overlays/eks/` include the component; `overlays/internal-services/` (the rehearsal child cluster — no cert-manager, no backup store) takes base alone.
- ArgoCD Applications point at an overlay, never at `base/` directly.

## Resource sizing

`base/kustomization.yaml` sets resources on the operator Deployment and `components/barman-cloud-plugin/kustomization.yaml` on the Barman Deployment. The numbers are derived from the production `cnpg-system` namespace's 30-day peak working-set memory in Prometheus, with headroom for the cluster-reconcile bursts that rate sampling doesn't capture. See README for the table.

When CNPG starts managing real Clusters and load grows, revisit these. The `manager` container's memory in particular will rise as a function of how many Cluster CRs it's reconciling.

## Verifying a render

```bash
kubectl kustomize --enable-helm overlays/oke
```

ArgoCD's repo-server runs kustomize with `--enable-helm`, so this matches what ArgoCD will apply. Without that flag the `helmCharts:` block is silently skipped and the render is incomplete.

## Upgrades

**Operator (Helm chart):** bump `helmCharts[0].version` in `base/kustomization.yaml`, verify the render, commit. After merge, bump the iak Application's `targetRevision`.

**Barman Cloud Plugin:** bump `tag` in `vendir.yml`, run `vendir sync`, verify the render, commit (including the regenerated `components/barman-cloud-plugin/vendor/`). After merge, bump the iak Application's `targetRevision`.

## When adding a new cluster

1. Create `overlays/<cluster>/kustomization.yaml` referencing `../../base`. Start with no patches.
2. Decide the plugin question explicitly: add `components: [../../components/barman-cloud-plugin]` only if the cluster runs cert-manager AND its databases will use Barman backups — omitting the component was the entire reason it exists (the internal-services child does neither).
3. If the cluster needs cluster-specific config (StorageClass, pull secrets, etc.), add it as a patch in that overlay.
4. Open a PR in `internal-applications-kub` adding the new ArgoCD Application + AppProject. AppProject name must equal the destination namespace per iak's policy.

## When the kustomize render emits a positional-patch error after an upgrade

Patches in `base/kustomization.yaml` use json-patch ops on the rendered output. If the upstream chart bumps and the JSON paths shift (especially args/env/volume indices), patches fail at build time with a clear error. Fix by re-pinning the patch path to the new index, ideally with an `op: test` guard before the `op: replace` so future bumps fail just as loudly.
