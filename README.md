# valheim-k8s

Kubernetes Helm chart for deploying a dedicated [Valheim](https://store.steampowered.com/app/892970/Valheim/) game server. Based on the community container image [`ghcr.io/community-valheim-tools/valheim-server`](https://github.com/community-valheim-tools/valheim-server-docker).

## Features

- **StatefulSet Architecture**: Deploys as a single-replica `StatefulSet` with rolling updates to avoid volume multi-attach contention on node failovers.
- **K8s-Native Persistence**: Defaults to `PersistentVolumeClaim` (PVC) for world data, server files, and dedicated backup storage, with support for pre-existing claims (`existingClaim`) and `hostvol` paths.
- **Dedicated Backup Storage & Management**: First-class support for a separate backup PVC, cron-based scheduling, idle detection, retention policies, and pre/post backup hooks for offsite sync.
- **Flexible Networking**: Supports `LoadBalancer` (with `loadBalancerClass`, static IP, and `externalTrafficPolicy: Local`), `NodePort`, and `hostNetwork`.
- **OCI Distribution**: Packaged and published directly to GitHub Container Registry (GHCR) as an OCI artifact.

## Installation

### Helm CLI (OCI)

```bash
helm install valheim-server oci://ghcr.io/evilhamsterman/valheim-k8s \
  --version 3.0.0 \
  --set worldName="my-world" \
  --set serverName="My Valheim Server" \
  --set password="supersecretpass"
```

### GitOps with Flux

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: evilhamsterman-charts
  namespace: flux-system
spec:
  type: oci
  interval: 1h
  url: oci://ghcr.io/evilhamsterman
---
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: valheim-server
  namespace: games
spec:
  interval: 1h
  chart:
    spec:
      chart: valheim-k8s
      version: "3.0.0"
      sourceRef:
        kind: HelmRepository
        name: evilhamsterman-charts
        namespace: flux-system
  values:
    worldName: "my-world"
    serverName: "My Valheim Server"
    passwordSecret: "valheim-credentials"
    networking:
      serviceType: LoadBalancer
      loadBalancerClass: "tailscale"
```

## Configuration

### World and Server Settings

| Parameter | Description | Default |
|:---|:---|:---|
| `worldName` | World name / save prefix | `example-world-name` |
| `serverName` | Name displayed in server browser | `example-server-name` |
| `password` | Server password (min. 5 characters) | `password` |
| `passwordSecret` | Name of existing Secret containing `password` key | `""` |
| `serverPublic` | List server in public browser | `true` |
| `serverArgs` | Additional command-line arguments to server | `""` |
| `image.repository` | Container image repository | `ghcr.io/community-valheim-tools/valheim-server` |
| `image.tag` | Container image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |

### Storage

Persistence is split into three separate volumes:

1. **`storage.data`**: World saves, admin/ban lists, and preferences (mounted to `/config`).
2. **`storage.server`**: Server binaries and Steam depot cache (mounted to `/opt/valheim`). Skipping re-downloads on restart.
3. **`storage.backups`**: Dedicated world backup archives (mounted to `/config/backups` or `backups.directory`).

| Parameter | Description | Default |
|:---|:---|:---|
| `storage.data.kind` | Storage type: `persistentVolumeClaim`, `hostvol`, or `emptyDir` | `persistentVolumeClaim` |
| `storage.data.pvc.size` | Size requested for world data PVC | `2Gi` |
| `storage.data.pvc.storageClassName` | StorageClass name (uses default if empty) | `""` |
| `storage.data.pvc.existingClaim` | Name of pre-existing PVC to bind | `""` |
| `storage.data.hostvol.path` | Host path when `kind: hostvol` | `/data/valheim` |
| `storage.server.kind` | Storage type: `persistentVolumeClaim`, `hostvol`, or `emptyDir` | `persistentVolumeClaim` |
| `storage.server.pvc.size` | Size requested for server binary PVC | `10Gi` |
| `storage.server.pvc.storageClassName` | StorageClass name (uses default if empty) | `""` |
| `storage.server.pvc.existingClaim` | Name of pre-existing PVC to bind | `""` |
| `storage.server.hostvol.path` | Host path when `kind: hostvol` | `/data/valheim-server` |
| `storage.backups.kind` | Storage type: `persistentVolumeClaim`, `hostvol`, `emptyDir`, or `""` (stores inside data volume) | `persistentVolumeClaim` |
| `storage.backups.pvc.size` | Size requested for backup volume | `10Gi` |
| `storage.backups.pvc.storageClassName` | StorageClass name (uses default if empty) | `""` |
| `storage.backups.pvc.existingClaim` | Name of pre-existing PVC to bind | `""` |
| `storage.backups.hostvol.path` | Host path when `kind: hostvol` | `/data/valheim-backups` |

### Backups

Managed via upstream container built-in backup engine:

| Parameter | Description | Default |
|:---|:---|:---|
| `backups.enabled` | Enable periodic automatic backups | `true` |
| `backups.cron` | Cron schedule expression for backups | `5 * * * *` |
| `backups.directory` | Container directory where backups are stored | `/config/backups` |
| `backups.maxAge` | Retention in days before pruning old backups | `3` |
| `backups.maxCount` | Maximum backup files kept (`0` for unlimited) | `0` |
| `backups.ifIdle` | Continue backups when no players are connected | `true` |
| `backups.idleGracePeriod` | Grace period in seconds to wait for final world save after last player disconnects | `3600` |
| `backups.zip` | Compress backups as `.zip` (`false` creates directories) | `true` |
| `backups.preBackupHook` | Shell command executed before backup runs (`@BACKUP_FILE@` placeholder) | `""` |
| `backups.postBackupHook` | Shell command executed after backup finishes (`@BACKUP_FILE@` placeholder) | `""` |

#### Offsite Backup Example

Use `extraVolumes` and `postBackupHook` to sync backups offsite via SCP/rsync/S3:

```yaml
extraVolumes:
  - name: backup-ssh-key
    type: secret
    defaultMode: 0600
    mountPath: /extraVolumes/backup-ssh-key

backups:
  postBackupHook: "timeout 300 scp -i /extraVolumes/backup-ssh-key/id_rsa -o StrictHostKeyChecking=no @BACKUP_FILE@ user@remote:~/backups/$(basename @BACKUP_FILE@)"
```

### Networking

| Parameter | Description | Default |
|:---|:---|:---|
| `useHostNetworking` | Run pod on host network namespace (bypasses Service) | `false` |
| `networking.serviceType` | Service type (`LoadBalancer`, `NodePort`, `ClusterIP`) | `LoadBalancer` |
| `networking.loadBalancerClass` | Optional LoadBalancer class implementation | `""` |
| `networking.loadBalancerIP` | Static IP for LoadBalancer | `""` |
| `networking.externalTrafficPolicy` | External traffic policy (`Local` preserves source IP, `Cluster`) | `""` |
| `networking.gamePort` | UDP port server listens on | `2456` |
| `networking.publishQueryPort` | Expose Steam query port (`gamePort + 1`) | `true` |
| `networking.nodePort` | Fixed port when using `serviceType: NodePort` | `30373` |
| `networking.serviceAnnotations` | Annotations for the Service | `{}` |
| `networking.serviceLabels` | Labels for the Service | `{}` |

### Pod Scheduling & Resources

| Parameter | Description | Default |
|:---|:---|:---|
| `resources` | Pod resource requests and limits | `{}` |
| `nodeSelector` | Node selector labels | `{}` |
| `tolerations` | Pod toleration rules | `[]` |
| `affinity` | Pod affinity rules | `{}` |
| `securityContext` | Pod security context | `{}` |
| `priorityClassName` | Priority class name | `""` |
| `podLabels` | Additional labels for pod | `{}` |
| `podAnnotations` | Additional annotations for pod | `{}` |
| `extraEnvironmentVars` | Additional environment variables for container | `{}` |
| `extraVolumes` | Additional volumes to mount into the pod | `[]` |

## Connecting to the Server

1. **Steam Favorites Browser** (Recommended):
   - In Steam, navigate to **View** -> **Game Servers** -> **Favorites**.
   - Click **+** and enter `<IP_OR_HOSTNAME>:<QUERY_PORT>` (default query port is `2457`, which is `gamePort + 1`).
2. **In-Game Browser**:
   - Search for `serverName` in the community/favorites list.
