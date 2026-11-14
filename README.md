# Concourse CI/CD on Podman

This repository contains the Quadlet configuration for running [Concourse CI](https://concourse-ci.org/) on Fedora using rootful Podman and systemd.

The deployment uses a Podman pod to group the Concourse components into a single application lifecycle. Quadlet generates the corresponding systemd units.

## Architecture

The deployment consists of:

- **Concourse web** — Web UI, API, pipeline scheduling, and TSA.
- **Concourse worker** — Executes build plans and tasks.
- **PostgreSQL** — Persistent Concourse database.
- **Concourse infra** — Infrastructure container created by Concourse as part of its worker runtime.

The containers run in the `concourse` Podman pod and share its network namespace. Consequently, services communicate over `localhost` rather than through separate container-network DNS names.

### Network ports

| Port | Purpose |
|---|---|
| 8080 | Concourse web UI and API |
| 9102 | Prometheus metrics |
| 2222 | TSA, internal to the pod |

Only ports 8080 and 9102 are published on the host. The worker connects to the web container's TSA service on `localhost:2222`.

## Storage

Persistent data is maintained in Podman named volumes:

| Volume | Container mount | Purpose |
|---|---|---|
| `concourse-db-data` | `/var/lib/postgresql` | PostgreSQL database |
| `concourse-web-state` | `/worker-state` | Concourse web state |
| `concourse-worker-state` | `/worker-state` | Worker state |

The volumes are managed by Podman and survive container restarts and recreation. They are created automatically when they do not already exist.

For rootful Podman, the default volume storage location is:

```text
/var/lib/containers/storage/volumes/
```

The deployment also mounts the host's `/etc/localtime` read-only into the containers so they use the host's local timezone.

## Configuration and keys

The quadlets provide default passwords (largely based on the Concourse web site) - update those to your local preferences.

Concourse's cryptographic keys are stored on the host under:

```text
/etc/concourse/keys/
├── web/
└── worker/
```

The web container uses the session-signing key, TSA host key, and authorized worker keys. The worker uses the TSA host public key and its own private key.

These files are mounted read-only only if explicitly configured that way; the current Quadlet mounts use SELinux relabeling (`:z`).

Keep private keys out of source control.

## Images

Container images are managed through separate Quadlet image units.

The current deployment uses:

- `docker.io/concourse/concourse:8.3.3`
- `docker.io/library/postgres:18.7`

Image versions are pinned in the Quadlet definitions rather than tracking a floating `latest` tag.

## Deployment

Quadlet files are deployed to:

```text
/etc/containers/systemd/
```

The files are managed in this repository and copied into that directory by the deployment process.

After updating the deployed Quadlet files, reload systemd's generated units:

```bash
systemctl daemon-reload
```

The application is managed through the pod unit:

```bash
systemctl start concourse-pod.service
systemctl stop concourse-pod.service
systemctl restart concourse-pod.service
```

Check the application status with:

```bash
systemctl status concourse-pod.service
systemctl status concourse-db.service
systemctl status concourse-web.service
systemctl status concourse-worker.service
```

Container status and logs can also be inspected directly with Podman:

```bash
podman pod ps
podman ps
podman logs concourse-web
podman logs concourse-worker
podman logs concourse-db
```

## Requirements

- Fedora Linux
- Rootful Podman with Quadlet support
- systemd
- Concourse-compatible persistent storage
- Concourse web and worker key material

This configuration is designed for a single-host Fedora deployment. It is not intended to be a portable, distribution-independent Podman configuration.

## Backups

The PostgreSQL database and Concourse state are held in named Podman volumes. Include these volumes in any backup strategy intended to preserve the deployment.

For a consistent database backup, use PostgreSQL's own backup facilities rather than copying a live database volume.

## Notes

Concourse pipelines and their configuration are managed separately from this repository. Pipeline creation and updates are automated outside the container deployment.
