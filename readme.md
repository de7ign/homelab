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
```

Retired storage domains: ~~minio.local~~ and ~~minio-console.local~~.

Retired project tracker domain: ~~youtrack.local~~.

## Local HTTPS behind Traefik
Itsaplan uses secure session cookies, so it needs HTTPS. Traefik serves one certificate on its `websecure` entrypoint, generated in WSL with [mkcert](https://github.com/FiloSottile/mkcert). The certificate only covers the hostnames listed on its SAN list, currently `itsaplan.local` and `api.itsaplan.local` — the only two routed through `websecure`/`tls=true` today.

Install mkcert in WSL first — it is not a Windows package and running it from PowerShell without this step fails:

```sh
sudo apt update && sudo apt install -y mkcert
```

Then, in WSL, from the repository root:

```sh
mkcert -install
mkdir -p infra/traefik/certs
mkcert -cert-file infra/traefik/certs/traefik.pem -key-file infra/traefik/certs/traefik-key.pem itsaplan.local api.itsaplan.local
mkcert -CAROOT
```

`mkcert -install` creates the root CA and only needs to run once. Note the path the last command prints — the next step needs it.

### Adding another host to the same certificate
1. Rerun the same `mkcert -cert-file ... -key-file ...` command above with the new hostname appended to the argument list; mkcert reissues `traefik.pem`/`traefik-key.pem` with every hostname you list, so include the existing ones too. Traefik's file provider watches the certificate files and reloads automatically — no restart needed.
2. In that service's `docker-compose.yml`, set its router's `entrypoints` label to `websecure` and add a `traefik.http.routers.<router>.tls=true` label, following the `itsaplan` and `itsaplan-api` routers in [apps/itsaplan/docker-compose.yml](apps/itsaplan/docker-compose.yml) as an example. Keep or add an HTTP-to-HTTPS redirect router if the service should no longer be reachable over plain HTTP.

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
docker compose --env-file apps/itsaplan/.env -f apps/itsaplan/docker-compose.yml up -d
```

Open Itsaplan at `https://itsaplan.local`.

RustFS replaces ~~MinIO~~ in the active setup; the MinIO compose file is retained but is not started.

Itsaplan replaces ~~Youtrack~~ in the active setup; the Youtrack compose file is retained but is not started.