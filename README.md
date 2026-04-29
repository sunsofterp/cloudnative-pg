# cloudnative-pg

CloudNativePG operator + Barman Cloud Plugin manifests, deployed via Kustomize overlays. The operator manages the full lifecycle of PostgreSQL clusters as native Kubernetes resources — replacing per-cluster RDS instances over time as part of the broader cloud-portability move.

## Layout

```
cloudnative-pg/
  base/                          # Cluster-agnostic rendered manifests
    namespace.yaml               # cnpg-system namespace
    operator.yaml                # helm template output, chart 0.26.1, app v1.27.1
    barman-cloud-plugin.yaml     # Barman Cloud Plugin 0.9.0 (S3 backups)
    kustomization.yaml           # plain resource list, no patches
  overlays/
    oke/                         # OKE deployment (no patches today)
      kustomization.yaml
```

The `base/` manifests are rendered from the upstream Helm chart and committed verbatim — same pattern as `ingress-nginx`. ArgoCD applies an overlay (not the base directly) so per-cluster differences can land as patches without forking the base.

## Overview

| Property | Value |
|----------|-------|
| Namespace | `cnpg-system` |
| Chart Version | 0.26.1 |
| Chart Source | https://cloudnative-pg.github.io/charts |
| Barman Plugin Version | 0.9.0 |
| Barman Plugin Source | https://github.com/cloudnative-pg/plugin-barman-cloud |

## Documentation

- [Official Documentation](https://cloudnative-pg.io/documentation/)
- [GitHub Repository](https://github.com/cloudnative-pg/cloudnative-pg)
- [Helm Chart](https://github.com/cloudnative-pg/charts)

## CRDs Provided

- `Cluster` — PostgreSQL cluster definition
- `Backup` — On-demand backup
- `ScheduledBackup` — Scheduled backups
- `Pooler` — PgBouncer connection pooling
- `ObjectStore` — Backup destination configuration (from Barman Cloud Plugin)

## Operator Upgrade Procedure

1. Check release notes: https://cloudnative-pg.io/documentation/current/release_notes/
2. Re-render the manifest:
   ```bash
   helm template cnpg cnpg/cloudnative-pg \
     --namespace cnpg-system \
     --version <NEW_VERSION> \
     --include-crds > base/operator.yaml
   ```
3. Commit and push. Open a PR; ArgoCD picks up the change after the iak Application's `targetRevision` is bumped to the new SHA.

## Barman Cloud Plugin Upgrade Procedure

1. Check release notes: https://github.com/cloudnative-pg/plugin-barman-cloud/releases
2. Download new manifest:
   ```bash
   VERSION=<NEW_VERSION>
   curl -sL "https://github.com/cloudnative-pg/plugin-barman-cloud/releases/download/v${VERSION}/manifest.yaml" \
     > base/barman-cloud-plugin.yaml
   ```
3. Commit and push. Open a PR; iak Application bump on merge.

## Related

- Architecture decisions documented in [keycloak-sso ADR-002](https://github.com/sunsofterp/keycloak-sso/blob/master/docs/decisions/ADR-002-database-architecture.md)
- Backup method rationale in [keycloak-sso ADR-010](https://github.com/sunsofterp/keycloak-sso/blob/master/docs/decisions/ADR-010-backup-method.md)
