## Must reads before diving in
- Create a shared network ONCE

```
docker network create proxy
```

- Youtrack needs and uses bind mounts instead of volumes/named volumes. Make sure to create mount directories and update the mounts accordingly in `/apps/youtrack/docker-compose.yml`

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
127.0.0.1   youtrack.local
127.0.0.1   minio.local
127.0.0.1   minio-console.local
127.0.0.1   grafana.local
127.0.0.1   homepage.local
```

## Startup Order
```
cd infra/traefik && docker compose up -d
cd infra/postgres && docker compose up -d
cd infra/minio && docker compose up -d
cd infra/redis && docker compose up -d

cd apps/youtrack && docker compose up -d
```