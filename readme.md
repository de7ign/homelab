## Must reads before diving in
- Create a shared network ONCE

```
docker network create proxy
```

> Note: Youtrack is retired in favour of itsaplan
- Youtrack needs and uses bind mounts instead of volumes/named volumes; this only matters if Youtrack is started again. Make sure to create mount directories and update the mounts accordingly in `/apps/youtrack/docker-compose.yml`

```
mkdir -p -m 750 <path to data directory> \
<path to logs directory> \
<path to conf directory> \
<path to backups directory>

chown -R 13001:13001 <path to data directory> \
<path to logs directory> \
<path to conf directory> \
<path to backups directory>
```

- Read this `https://www.jetbrains.com/help/youtrack/server/upgrade-with-docker-image.html#upgrading-docker-image` before changing Youtrack image version

- Update hosts to resolve traefik domains
```
Edit C:\Windows\System32\drivers\etc\hosts as an administrator

Add your domains
127.0.0.1   rustfs.local
127.0.0.1   rustfs-console.local
127.0.0.1   itsaplan.local
127.0.0.1   api.itsaplan.local
127.0.0.1   dbpro.local
```

Retired storage domains: ~~minio.local~~ and ~~minio-console.local~~.

Retired project tracker domain: ~~youtrack.local~~.

## Local HTTPS behind Traefik
Itsaplan and DB Pro use secure session cookies, so they need HTTPS. Traefik serves one certificate on its `websecure` entrypoint, generated in WSL with [mkcert](https://github.com/FiloSottile/mkcert). The certificate must include all HTTPS hostnames, including `dbpro.local`; reissue an existing certificate if it does not.

Install mkcert in WSL first — it is not a Windows package and running it from PowerShell without this step fails:

```sh
sudo apt update && sudo apt install -y mkcert
```

Then, in WSL, from the repository root:

```sh
mkcert -install
mkdir -p infra/traefik/certs
mkcert -cert-file infra/traefik/certs/traefik.pem -key-file infra/traefik/certs/traefik-key.pem itsaplan.local api.itsaplan.local dbpro.local
mkcert -CAROOT
```

`mkcert -install` creates the root CA and only needs to run once. Note the path the last command prints — the next step needs it.

### Adding another host to the same certificate
1. Rerun the same `mkcert -cert-file ... -key-file ...` command above with the new hostname appended to the argument list; mkcert reissues `traefik.pem`/`traefik-key.pem` with every hostname you list, so include the existing ones too. Traefik's file provider watches the certificate files and reloads automatically — no restart needed.
2. In that service's Compose config, set its router's `entrypoints` label to `websecure` and add a `traefik.http.routers.<router>.tls=true` label. Itsaplan keeps these Traefik-specific settings in [apps/itsaplan/docker-compose.traefik.yml](apps/itsaplan/docker-compose.traefik.yml), separate from the base stack. Keep or add an HTTP-to-HTTPS redirect router if the service should no longer be reachable over plain HTTP.

The leaf certificate and key are ignored by Git and never leave WSL.

Trust the root CA in Windows so browsers there accept the certificate. This requires WSL itself to be installed under your regular, non-admin Windows user — `\\wsl.localhost\...` paths are mapped only in that interactive session, so an elevated PowerShell session, or WSL installed under a different account, can't resolve them. Copy the file to a local Windows path first, then trust it from an elevated session, and remove the local copy afterwards:

```powershell
# Non-admin PowerShell, replace <user> with your WSL username
New-Item -ItemType Directory "$env:LOCALAPPDATA\TempCert" -Force

Copy-Item `
  "\\wsl.localhost\Ubuntu\home\<user>\.local\share\mkcert\rootCA.pem" `
  "$env:LOCALAPPDATA\TempCert\rootCA.pem"
```

```powershell
# Admin PowerShell, replace <windows-user> with your Windows username
certutil -addstore -f "ROOT" "C:\Users\<windows-user>\AppData\Local\TempCert\rootCA.pem"
```

```powershell
# Non-admin PowerShell again — the local copy isn't needed once trusted
Remove-Item "$env:LOCALAPPDATA\TempCert\rootCA.pem"
```

Only the root certificate crosses into Windows; its private key stays in WSL. Firefox keeps its own trust store on Windows, so importing into `about:preferences#privacy` > Certificates is needed separately if you use it.

## Environment variables
Copy `.env.example` to `.env` and set strong credentials before starting services. `.env` is local-only and should never be committed.

## Startup Order
Run these commands from the repository root:
```
docker compose --env-file .env -f infra/traefik/docker-compose.yml up -d
docker compose --env-file .env -f infra/postgres/docker-compose.yml up -d
docker compose --env-file .env -f infra/rustfs/docker-compose.yml up -d
docker compose --env-file .env -f infra/redis/docker-compose.yml up -d
docker compose --env-file apps/itsaplan/.env -f apps/itsaplan/docker-compose.yml -f apps/itsaplan/docker-compose.traefik.yml up -d
docker compose -f apps/dbpro/docker-compose.yml up -d
```

Open Itsaplan at `https://itsaplan.local`.

Open DB Pro at `https://dbpro.local` and create the admin account on first visit. Restrict access until that account is created. DB Pro's `/data` volume contains its SQLite database and generated encryption key; back up both together, since losing the key makes saved connection credentials unreadable. The shared PostgreSQL and Redis services are reachable from DB Pro as `postgres:5432` and `redis:6379` on the `proxy` network. Itsaplan's separate database is not on that network.

RustFS replaces ~~MinIO~~ in the active setup; the MinIO compose file is retained but is not started.

Itsaplan replaces ~~Youtrack~~ in the active setup; the Youtrack compose file is retained but is not started.

## Migrating the shared PostgreSQL service
PostgreSQL major versions cannot use each other's data directories directly. For a major-version upgrade, use a dump and restore as below, or use `pg_upgrade`. These instructions apply to the shared service in [infra/postgres/docker-compose.yml](infra/postgres/docker-compose.yml), not Itsaplan's separate PostgreSQL service and volume.

Run the commands from the repository root. Stop clients before dumping and keep them stopped until the restored data is verified. The source server must still be running, and the target must use a **new, empty volume**. Keep the original Compose configuration and volume for rollback; do not run `docker compose down -v`.

The dump includes databases, roles and data, including password hashes; protect it like a secret. It does not include server configuration or extension binaries. Check target-version compatibility, install any required extensions, and account for custom tablespaces and locale/collation requirements before restoring. This procedure is for upgrades, not downgrades.

Stop if any command fails. A failed restore can leave partial data in the target volume; investigate before retrying. Do not replace the original volume or discard the dump until the restore has been verified.

### 1. Dump the source cluster
The temporary bootstrap role and database name `homelab_pg_migrate` must not already exist in the source cluster. Check roles and databases with `\du` and `\l` in `psql`; if either name exists, choose another unique name and replace it consistently in the commands below.

Create a private backup directory and dump the cluster. `BACKUP_DIR` must remain set in the same shell for the restore step. Check that `pg_dumpall` succeeds before proceeding; a nonempty file alone does not prove a successful dump.

```sh
umask 077
BACKUP_DIR=$(mktemp -d "$HOME/postgres-migration.XXXXXXXX")
echo "Backup directory: $BACKUP_DIR"
docker exec postgres sh -c 'pg_dumpall -U "$POSTGRES_USER"' > "$BACKUP_DIR/all.sql"
test -s "$BACKUP_DIR/all.sql"
```

### 2. Start the new version
Stop the source server:

```sh
docker compose --env-file .env -f infra/postgres/docker-compose.yml stop postgres
```

Update the Compose file to use the target image and a new volume name, leaving the old volume unattached. Ensure the target volume name does not already refer to an existing volume. Follow the target image's documentation for the correct mount path; for the official PostgreSQL 18 image, mount the new volume at `/var/lib/postgresql`, not the old `/var/lib/postgresql/data` path. Never attach the old data volume to the new server.

Start the target with a temporary bootstrap superuser, which avoids conflicting with roles restored from the dump:

```sh
POSTGRES_USER=homelab_pg_migrate docker compose --env-file .env -f infra/postgres/docker-compose.yml up -d postgres
docker exec postgres pg_isready -h 127.0.0.1 -U homelab_pg_migrate -d postgres
```

If `pg_isready` reports that the server is not ready, wait and retry; inspect `docker logs postgres` if it does not become ready.

### 3. Restore and verify
Restore the dump into the new server:

```sh
docker exec -i postgres sh -c 'psql -X -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d postgres' \
  < "$BACKUP_DIR/all.sql"
```

Recreate the container using the regular `POSTGRES_USER` from `.env`, wait for it to be ready, then remove the temporary database and disable the temporary bootstrap role:

```sh
docker compose --env-file .env -f infra/postgres/docker-compose.yml up -d --force-recreate postgres
docker exec postgres pg_isready -h 127.0.0.1
docker exec postgres sh -c 'psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" -d postgres -c "DROP DATABASE IF EXISTS homelab_pg_migrate" -c "ALTER ROLE homelab_pg_migrate NOLOGIN PASSWORD NULL"'
```

The bootstrap role cannot be dropped because PostgreSQL requires it internally; it remains a superuser with login disabled and no password. Verify the restored databases and application data before restarting clients. Retain the dump and original volume until satisfied; to roll back before accepting new writes, stop the target and restore the original Compose configuration and volume.
