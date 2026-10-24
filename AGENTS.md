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
- Dedicated backup PVC mounts over `/config/backups` unless `storage.backups.kind` is empty (`""`), in which case backups write to the primary world data volume.

## Local Verification & Testing

Always test chart changes locally before pushing:

```bash
# Lint the chart
helm lint ./chart

# Test template rendering with default values
helm template valheim ./chart

# Test with specific configurations
helm template valheim ./chart \
  --set networking.loadBalancerClass=tailscale \
  --set storage.backups.pvc.existingClaim=custom-backup-pvc
```

## Git & Worktree Conventions

- This repository uses git worktrees. The `main` directory must stay on `main`.
- For new feature branches, always create a new worktree (`git worktree add ../<branch-name> <branch-name>`), unless Dan explicitly instructs to commit directly to `main`.
- Before pushing, run `git fetch origin && git log origin/main..HEAD` to confirm branch status.

## Release & CI/CD Workflow

- Version number is tracked in both `version` and `chart/Chart.yaml` (keep both in sync).
- CI workflow (`.github/workflows/deploy-chart.yml`) triggers on pushes to `main` touching `chart/**` or `version`.
- Packages the chart with Helm and pushes the OCI package to `oci://ghcr.io/${OWNER}`.
