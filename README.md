# Docker application stacks

Dockhand deploys these stacks from the `main` branch. Each top-level directory
is an independent Git stack whose compose path is `<stack>/docker-compose.yml`.
Leave Dockhand's Context directory unset so a change only redeploys that stack.

Pangolin, Newt, Dockhand, Hawser, and Pocket-ID are control-plane services and
do not belong in this repository. They are deployed with Ansible.

## Create a stack in Dockhand

1. Add this repository once and select branch `main`.
2. Create a Git stack in the target Dockhand environment.
3. Set its compose path, for example `paperlessngx/docker-compose.yml`.
4. Copy the stack's `.env.example` keys into Dockhand.
5. Mark passwords, tokens, client secrets, and encryption keys as secrets.
6. Validate, then deploy.

Dockhand environments represent sites. Deploying fewer services at a site
means omitting those Git stacks; it does not require a site branch.

For a new site, deploy `storage-init/docker-compose.yml` once before application
stacks. It creates the known `${DATA_ROOT}` directories with the site's
`PUID:PGID`, then exits. Site-specific mounts outside `DATA_ROOT` (media,
consume, notes, and similar) must already exist with suitable permissions.

## Required conventions

- `SITE_ID` is a stable, Pangolin-safe site identifier such as `sqml01-rs`.
- `DATA_ROOT` is the site's persistent data root (`/opt/docker`,
  `/volume1/docker`, or another host path).
- `OIDC_ISSUER` is the external Pocket-ID issuer for that site.
- `PUID`, `PGID`, `TZ`, domains, and image tags are site/stack overrides.
- `./config` is only for static configuration shipped beside the compose file.
- Persistent data uses `${DATA_ROOT}` or named volumes.
- Cross-stack mounts use explicit variables or `${DATA_ROOT}`, never `../`.
- Secrets are never committed.

Hawser deliberately starts Compose with a clean environment. Setting
`DATA_ROOT`, `SITE_ID`, or `OIDC_ISSUER` on the Hawser container does not make
them available to stacks. Add the common site values to every Git stack's
Dockhand environment panel (a Dockhand config set can seed new stacks, but
later config-set changes are not propagated automatically).

Pangolin resource label keys include `${SITE_ID}` to avoid collisions when
multiple sites share one Pangolin organization. Homepage labels remain on each
public service. Pangolin must have Docker socket/container-label discovery
enabled for each Newt site, otherwise these labels are not reconciled.

## Hawser paths

Hawser receives the compose directory and writes it under its `STACKS_DIR`.
The host and container path must match, for example:

```yaml
volumes:
  - /opt/hawser-stacks:/opt/hawser-stacks
environment:
  STACKS_DIR: /opt/hawser-stacks
```

`STACKS_DIR` is separate from `DATA_ROOT`. Do not mix transferred stack files
with application databases, media, or backups.

## Shared networks

Ansible creates these before Dockhand deploys apps:

- `pangolin` for Newt discovery and public resources
- `db_network` for shared PostgreSQL/Redis consumers

Deployment order:

1. `storage-init`
2. `postgres16` and `redis7`
3. application stacks
