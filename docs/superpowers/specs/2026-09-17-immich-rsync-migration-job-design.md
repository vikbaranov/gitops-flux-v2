# Immich Library Rsync Migration Job — Design

Date: 2026-09-17
Status: Approved

## Purpose

A one-off Kubernetes Job that copies an existing photo library from a single
remote host into the `immich-library-pvc` PersistentVolumeClaim (namespace
`immich`) so Immich can serve it, using rsync over SSH with key authentication.

## Requirements

- Mount the existing `immich-library-pvc` (200Gi, topolvm-provisioner, RWO).
- Pull data from a single remote host/path via `rsync` over SSH.
- Authenticate with an SSH private key stored in a manually created Secret;
  strict host key checking stays enabled via a `known_hosts` entry in the same
  Secret.
- Manifest is standalone: committed to the repo for reference but not
  referenced by any Kustomization, so Flux never reconciles or re-triggers it.
  Applied and deleted manually with `kubectl`.

## Design

### File placement

`apps/immich/job/rsync-migration-job.yaml` — the `job/` directory sits outside
`base/` and `overlays/`, and no Kustomization references it.

### Secret (out of git)

`immich-rsync-ssh` in namespace `immich`, created manually:

```
kubectl create secret generic immich-rsync-ssh -n immich \
  --from-file=id_ed25519=/path/to/private/key \
  --from-file=known_hosts=/path/to/known_hosts
```

(`known_hosts` can be produced with `ssh-keyscan -H <host> > known_hosts`.)

### Job

- Image: `alpine:3.22` (pinned minor, renovate-annotated following repo
  convention). The script installs `rsync` and `openssh-client` with
  `apk add --no-cache` at start.
- Configuration via three environment variables at the top of the manifest —
  the only values the operator edits: `SRC_HOST`, `SRC_PATH`, `SRC_USER`.
- Volumes: PVC mounted at `/mnt/library`; Secret mounted read-only at
  `/mnt/ssh` with `defaultMode: 0600`.
- Transfer command:

  ```
  rsync -aHAX --partial --info=stats2,progress2 \
    -e "ssh -i /mnt/ssh/id_ed25519 -o UserKnownHostsFile=/mnt/ssh/known_hosts \
        -o StrictHostKeyChecking=yes -o BatchMode=yes" \
    "$SRC_USER@$SRC_HOST:$SRC_PATH/" /mnt/library/
  ```

  - `-aHAX`: permissions, times, symlinks, hard links, ACLs, xattrs.
  - Trailing slash on the source path: contents land in the PVC root.
  - `--partial`: a retried pod resumes interrupted files instead of
    restarting them.
  - `BatchMode=yes`: fail fast rather than hang on a password prompt.
  - No `--delete`: the job never removes anything already in the PVC.
- `restartPolicy: OnFailure`, `backoffLimit: 3` — rsync is idempotent, so
  retries are safe.
- Runs as root (required for `apk` and for writing the topolvm volume).
  Acceptable for a one-off migration job.
- Resources: requests `100m` CPU / `256Mi` memory, no limits (the initial
  file scan of a large library should not be throttled).

## Verification

1. `kubectl apply --dry-run=server -n immich -f apps/immich/job/rsync-migration-job.yaml`
   before the real apply.
2. `kubectl logs -f -n immich job/immich-rsync-migration` to watch progress.
3. Compare `du` totals on source and PVC after completion.

## Out of scope

- Multiple source hosts, per-host subdirectories, or merged layouts.
- Recurring syncs (CronJob) — the manifest can be converted later if needed.
- Flux automation, sealed secrets, or CI for this Job.
