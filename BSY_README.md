# BSY CTFd Deployment

This fork of CTFd adds two custom Docker Compose configurations for running the
BSY CTF platform: **production** (`prod`) and **staging**.

## Starting the services

The two environments run as **separate Compose projects/namespaces**, so they can
be started, stopped, and inspected independently.

### Production

```
docker compose -f docker-compose_prod.yml -p prod up -d
```

### Staging

```
docker compose -f docker-compose_staging.yml -p staging up -d
```

> `-p prod` / `-p staging` sets the Compose **project name**. This is what prefixes
> your containers (e.g. `prod-ctfd`, `staging-ctfd-staging`) and keeps the two
> environments isolated from one another (separate networks, volumes, container names).

Stop / tear down a specific environment with the same project name, e.g.:

```
docker compose -f docker-compose_prod.yml -p prod down
```

### Rebuilding the image

When rebuilding the image, the `.prod_data` / `.staging_data` directories may be
**owned by root** (containers run as `user: root`). Because these directories are
bind-mounted, Docker will fail to build the image while they are in place.

To build successfully, temporarily move the data directory aside, build, then
restore it:

```
mv .prod_data .prod_data.bak

docker compose -f docker-compose_prod.yml build

# Either restore the previous state...
mv .prod_data.bak .prod_data

# ...or start fresh (CTFd will reinitialize) and restore from a backup/re-import as needed
rm -rf .prod_data.bak
docker compose -f docker-compose_prod.yml -p prod up -d
```

(The same applies to `.staging_data` with the staging compose file.)

### Environment files & data

Each environment expects its own `.env` file and produces its own local data
directory (all git-ignored):

| Environment | Env file       | Data directory   |
|-------------|----------------|------------------|
| prod        | `.bsy_env`     | `.prod_data/`    |
| staging     | `.staging_env` | `.staging_data/` |

These hold secrets (DB credentials, Traefik/notifier tokens, etc.) and persistent
data (MySQL, Redis, CTFd logs/uploads), respectively.

## Traefik labels

Both compose files attach their `ctfd` / `ctfd-staging` service to an **external
Traefik** network and expose routing rules through **container labels**. Traefik
discovers the services from these labels (no per-container Traefik config needed).

The labels in each file do the following:

- `traefik.enable=true`
  - Explicitly tells Traefik to route to this container (so it doesn't skip it).
- `traefik.http.routers.<name>.entrypoints=websecure`
  - Attaches the router to the `websecure` entrypoint (HTTPS, port 443).
- `traefik.http.routers.<name>.rule=Host(\`<hostname>\`)`
  - Defines the hostname that routes to this container, e.g.
    `ctfd.bsy.fel.cvut.cz` (prod) or `ctfd-staging.stratosphereips.org` (staging).
- `traefik.http.routers.<name>.tls.certresolver=resolver`
  - Enables TLS for this router and uses the named `resolver` (`resolver`) to
    automatically issue/renew Let's Encrypt certificates.
- `traefik.http.services.<name>.loadbalancer.server.port=8000`
  - Tells Traefik the container port to forward traffic to (CTFd listens on 8000).
- `traefik.docker.network=<external_network>`
  - Specifies which Docker network Traefik shares with the container
    (`bsy_ctfd` for prod, `bsy_ctfd_staging` for staging), so it can reach it.

The `<name>` in the labels is the router/service name scoped to the project, e.g.
`ctfd` (prod) and `ctfd-staging` (staging). The external networks
(`bsy_ctfd`, `bsy_ctfd_staging`) and Traefik itself are managed outside of these
compose files.

## Networks

Each compose file defines two networks for its stack:

- **external network** (`bsy_ctfd` / `bsy_ctfd_staging`): shared with Traefik for
  inbound HTTPS traffic.
- **internal network** (`internal`): a private, non-external network connecting
  only the CTFd, database, and cache services (so the DB/Redis are not exposed).
