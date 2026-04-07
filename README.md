# ash-twin-monitoring

A self-contained **ELK + Filebeat** stack that collects **all Docker container logs** on your VPS and exposes them through a single Kibana web UI.

- **Elasticsearch** – stores and indexes log data  
- **Filebeat** – scrapes every container's logs automatically (no per-project changes needed)  
- **Kibana** – browser-accessible dashboard at `http://<your-vps-ip>:5601`

Because Filebeat uses Docker autodiscovery, any container you deploy later — from any other `docker compose` project — is picked up automatically without any extra configuration.

---

## Table of contents

1. [Prerequisites](#1-prerequisites)  
2. [Deployment](#2-deployment)  
3. [Accessing Kibana](#3-accessing-kibana)  
4. [Creating a data view in Kibana](#4-creating-a-data-view-in-kibana)  
5. [How it works](#5-how-it-works)  
6. [Filtering logs by container or project](#6-filtering-logs-by-container-or-project)  
7. [Configuration reference](#7-configuration-reference)  
8. [Opt-out a container](#8-opt-out-a-container)  
9. [Troubleshooting](#9-troubleshooting)  

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| Docker ≥ 20.10 | `docker --version` |
| Docker Compose plugin ≥ 2.x | `docker compose version` |
| Docker context pointing at your VPS | `docker context use myvps` |
| TCP 5601 open in your VPS firewall | Kibana web UI |
| TCP 9200 open (optional) | Elasticsearch direct access |
| ≥ 2 GB RAM on the VPS | Elasticsearch needs at least 512 MB heap |

---

## 2. Deployment

```bash
# 1. Clone this repository onto your local machine
git clone https://github.com/DAnikeyev/ash-twin-monitoring.git
cd ash-twin-monitoring

# 2. (Optional) Tune settings in .env
#    e.g. increase ES_JAVA_OPTS if you have more RAM available
nano .env

# 3. Point Docker at your VPS
docker context use myvps

# 4. Start the stack (detached)
docker compose up -d
```

Docker will pull the images and start three containers:

| Container | Image | Port |
|---|---|---|
| `elasticsearch` | `docker.elastic.co/elasticsearch/elasticsearch` (version from `.env`) | 9200 |
| `kibana` | `docker.elastic.co/kibana/kibana` (version from `.env`) | **5601** |
| `filebeat` | `docker.elastic.co/beats/filebeat` (version from `.env`) | – |

Kibana takes ~30–60 seconds to become ready after Elasticsearch is healthy.

---

## 3. Accessing Kibana

Open your browser and navigate to:

```
http://<your-vps-ip>:5601
```

Replace `<your-vps-ip>` with the public IP address of your VPS (e.g. `http://203.0.113.42:5601`).

> **Tip:** If you use a domain name, point an A record at your VPS IP and update `SERVER_PUBLICBASEURL` in `docker-compose.yml` accordingly.

---

## 4. Creating a data view in Kibana

Filebeat automatically creates an Elasticsearch index template. You only need to create a **Data View** once after first deploy:

1. Open Kibana → **☰ Menu → Stack Management → Data Views**  
2. Click **Create data view**  
3. Set **Index pattern** to `docker-logs-*`  
4. Set **Timestamp field** to `@timestamp`  
5. Click **Save data view to Kibana**  
6. Go to **☰ Menu → Discover** to start exploring logs  

---

## 5. How it works

```
┌─────────────────────────────────────────────────────┐
│                        VPS                          │
│                                                     │
│  ┌──────────────┐   ┌──────────────┐               │
│  │  Project A   │   │  Project B   │  ... any       │
│  │  container 1 │   │  container 3 │      project   │
│  │  container 2 │   │  container 4 │                │
│  └──────┬───────┘   └──────┬───────┘               │
│         │ stdout/stderr     │                        │
│         ▼                  ▼                        │
│  /var/lib/docker/containers/**/*.log                │
│         │                                           │
│         ▼                                           │
│  ┌─────────────┐    ┌─────────────────┐            │
│  │  Filebeat   │───▶│  Elasticsearch  │            │
│  │(autodiscover│    │  docker-logs-*  │            │
│  │ all containers)  └────────┬────────┘            │
│  └─────────────┘             │                     │
│                               ▼                     │
│                        ┌──────────┐                 │
│                        │  Kibana  │◀── browser       │
│                        │  :5601   │                 │
│                        └──────────┘                 │
└─────────────────────────────────────────────────────┘
```

**Filebeat** runs with `user: root` and mounts:
- `/var/lib/docker/containers` (read-only) – raw JSON log files written by Docker  
- `/var/run/docker.sock` (read-only) – to query container metadata  

It uses **Docker autodiscovery**: when any container starts anywhere on the host, Filebeat automatically begins tailing its log file. Each log event is enriched with metadata like container name, image, labels and the Docker Compose project name.

---

## 6. Filtering logs by container or project

In **Kibana → Discover**, use the search bar or **Add filter** to narrow down logs. Useful fields enriched by Filebeat:

| Field | Example value | Description |
|---|---|---|
| `container.name` | `my-api` | Container name |
| `container.image.name` | `nginx:alpine` | Docker image |
| `docker.container.labels.com.docker.compose.project` | `myproject` | Compose project name |
| `docker.container.labels.com.docker.compose.service` | `web` | Compose service name |
| `host.name` | `myvps` | VPS hostname |
| `log.level` | `error` | Log level (if parsed) |

**Example KQL queries in Discover:**

```
# All logs from a specific container
container.name: "my-api"

# All logs from a Compose project
docker.container.labels.com.docker.compose.project: "myproject"

# Error logs across all containers
message: *error* or message: *ERROR*

# Logs from a specific image
container.image.name: "nginx*"
```

---

## 7. Configuration reference

All tuneable values live in `.env`:

| Variable | Default | Description |
|---|---|---|
| `ELASTIC_VERSION` | `8.12.2` | Version of all three images |
| `ES_JAVA_OPTS` | `-Xms512m -Xmx512m` | Elasticsearch JVM heap (increase for heavy load) |
| `KIBANA_PORT` | `5601` | Host port for Kibana |
| `ELASTICSEARCH_PORT` | `9200` | Host port for Elasticsearch |

Edit `.env` and run `docker compose up -d` again to apply changes.

---

## 8. Opt-out a container

To stop Filebeat from collecting logs from a specific container, add this label to that container's service in its own `docker-compose.yml`:

```yaml
labels:
  co.elastic.logs/enabled: "false"
```

---

## 9. Troubleshooting

**Kibana shows "No results" in Discover**

- Wait 1–2 minutes after startup for Filebeat to ship the first batch.  
- Check Filebeat is running: `docker compose logs filebeat`  
- Check Elasticsearch is healthy: `curl http://<vps-ip>:9200/_cluster/health`  

**Filebeat permission errors**

Filebeat requires read access to `/var/lib/docker/containers`. The `user: root` setting in `docker-compose.yml` handles this. If you see `permission denied`, verify the host Docker data directory matches `/var/lib/docker/containers`.

**Out of disk space**

Indices grow over time. Consider adding an ILM (Index Lifecycle Management) policy in Kibana (**Stack Management → Index Lifecycle Policies**) to delete old indices automatically.

**Updating to a newer ELK version**

Change `ELASTIC_VERSION` in `.env` and run:

```bash
docker compose pull
docker compose up -d
```

All three images must be on the same version.

---

## Stopping the stack

```bash
docker compose down          # stop, keep data volumes
docker compose down -v       # stop AND delete all data
```
