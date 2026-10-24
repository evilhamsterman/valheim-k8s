# AGENTS.md

Instructions and context for AI coding agents working in the `valheim-k8s` repository.

## Repository Overview

`valheim-k8s` is a Helm chart repository for running dedicated Valheim servers on Kubernetes.
- **Upstream Image**: `ghcr.io/community-valheim-tools/valheim-server` ([community-valheim-tools/valheim-server-docker](https://github.com/community-valheim-tools/valheim-server-docker))
- **Distribution**: Published as an OCI package to GitHub Container Registry at `oci://ghcr.io/evilhamsterman/valheim-k8s`.
- **Target Environments**: Kubernetes clusters (homelab & enterprise), frequently managed via Flux GitOps.

## Chart Architecture & Templates

- **`chart/templates/statefulset.yaml`**: Deploys the server as a single-replica `StatefulSet` with `RollingUpdate` strategy to guarantee single-pod access to RWO storage and stable network identity.
- **`chart/templates/persistentvolumeclaim.yaml`**: Creates PersistentVolumeClaims for:
  - World data: `/config` (`{{ .Release.Name }}-world-data`)
  - Server installation cache: `/opt/valheim` (`{{ .Release.Name }}-server-base-data`)
  - Backups: `/config/backups` (`{{ .Release.Name }}-backup-data`)
  - All PVCs support `.existingClaim` to bind pre-existing volumes and bypass manifest creation.
- **`chart/templates/service.yaml`**: Exposes UDP `gameport` (default 2456) and `queryport` (default 2457). Supports `serviceType: LoadBalancer` with `loadBalancerClass`, `loadBalancerIP`, and `externalTrafficPolicy`.
- **`chart/values.yaml`**: Chart configuration defaults. Storage defaults to `persistentVolumeClaim` with `hostvol` as a fallback.

## Backup Management

Backups are powered by the upstream container's backup daemon:
- Built-in options in `values.yaml` under `backups:` map directly to container environment variables (`BACKUPS`, `BACKUPS_CRON`, `BACKUPS_DIRECTORY`, `BACKUPS_MAX_AGE`, `BACKUPS_MAX_COUNT`, `BACKUPS_IF_IDLE`, `BACKUPS_IDLE_GRACE_PERIOD`, `BACKUPS_ZIP`, `PRE_BACKUP_HOOK`, `POST_BACKUP_HOOK`).
- Dedicated backup volume mounts over `/config/backups` configured via `backups.storage`. Setting `backups.enabled: true` enables both the container backup engine and backup storage, requiring valid `backups.storage` configuration. Setting `backups.enabled: false` disables backups and omits the backup volume.

## Local Verification & Testing

A `Taskfile.yml` is provided for common development workflows (run via `task` or `go-task`):

```bash
# Lint the chart
task lint

# Render templates locally
task template

# Package the chart for OCI release
task build

# Deploy or upgrade directly from local repo
task deploy -- --values my-values.yaml

# Check status or clean build artifacts
task status
task clean
```

## CI/CD Workflows

- **CI Test (`.github/workflows/ci.yml`)**: Runs on `push` and `pull_request` to `main` touching chart files, Taskfile, or version. Executes `task lint`, `task build` (test packaging), and `task template`.
- **Release (`.github/workflows/deploy-chart.yml`)**: Runs on published GitHub releases (and manual `workflow_dispatch`). Packages the chart and pushes the OCI artifact to `oci://ghcr.io/${OWNER}`.
- Version number is tracked in both `version` and `chart/Chart.yaml` (keep both in sync).
