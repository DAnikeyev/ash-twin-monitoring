## Quick cheat sheet — common commands

This file contains the minimal commands to run the stack. All configuration (including secrets) lives in a single `.env` file, which is git-ignored. The repository includes `.env.example` as a template.

1) Prepare local config (one-time)

```powershell
Set-Location "C:\Repos\GitHub\ash-twin-monitoring"
Copy-Item .env.example .env
notepad .env    # edit and fill ELASTIC_PASSWORD, KIBANA_LOGIN_PASSWORD, etc.
```

2) Start the stack

```powershell
docker compose up -d --build
```

3) Watch the one-shot user creation (kibana-user-setup)

```powershell
docker compose logs kibana-user-setup --follow
```

4) Useful diagnostics

```powershell
docker compose ps
docker compose logs kibana --tail=200
docker compose logs elasticsearch --tail=200
docker compose logs filebeat --tail=200
```

5) Verify Kibana UI / login (from the VPS host)

```powershell
# Replace with the Kibana UI username you set in .env
curl.exe -u login:password http://127.0.0.1:9200/_security/_authenticate
```

6) Stop / remove

```powershell
# stop (keep volumes)
docker compose down

# stop and delete volumes (data loss)
docker compose down -v

# stop and also remove images (full cleanup)
docker compose down --rmi all
```

Tips
- `.env` is git-ignored — keep it local.
- If you change `.env`, run `docker compose up -d` again to pick up changes.
- If Filebeat or Kibana fail to register templates, wait ~30–60 seconds after Elasticsearch is healthy so Filebeat can push templates.

## Resource limits

Each container has a Docker memory cap (set in `docker-compose.yml`):

| Container       | `mem_limit` |
|-----------------|-------------|
| elasticsearch   | 1536 MB     |
| kibana          | 1024 MB     |
| filebeat        | 256 MB      |

Elasticsearch heap is additionally capped by `ES_JAVA_OPTS` in `.env` (default `-Xms512m -Xmx512m`).

## Log retention (ILM)

An ILM policy (`docker-logs-policy`) auto-deletes log indices older than **30 days**. It is created once via `scripts/setup-ilm.sh` and attached to the `docker-logs` index template.

To re-run after a fresh Elasticsearch (e.g. after `docker compose down -v`):

```bash
# On the VPS
bash /opt/ash-twin/scripts/setup-ilm.sh
```

To change retention, edit the `min_age` value in `scripts/setup-ilm.sh` and re-run it.

The monitoring stack's own containers (elasticsearch, kibana, filebeat) are excluded from log collection via the `co.elastic.logs/enabled: "false"` Docker label to avoid a feedback loop.

