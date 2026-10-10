# Local Docker services

Run commands from the repository root. Image references are centralized in
`docker-compose.yml` under `x-images`; service configuration lives in
`docker/<service>/`, and persistent files live in `volumes/<service>/`.
Docker itself stores image layers in its managed image store, not these bind mounts.

Copy `.env.example` to `.env` if needed and replace the Langfuse secret placeholders.
PostgreSQL and MinIO settings are shared with all their consumers. Credentials used
in connection URLs must be URL-safe. Published ports accept local connections only.

## Start services as needed

```sh
docker compose config --quiet
docker compose up -d --wait mysql redis
docker compose up -d --wait langfuse-web langfuse-worker
docker compose up -d --wait milvus
docker compose up -d --wait nginx prometheus grafana
docker compose up -d --wait mongo postgres kafka
docker compose up -d ubuntu22 ubuntu24
docker compose exec ubuntu22 bash
docker compose exec ubuntu24 bash
```

Ubuntu 22.04 (`ubuntu22`) and Ubuntu 24.04 (`ubuntu24`) belong to the `tools` profile;
naming them explicitly starts them. `docker compose up -d --wait` starts all other
services, including the heavier tracing and vector workloads. Choose services
explicitly on a small machine.

`mc` and `milvus-volume-init` are initialization jobs, not additional servers. They
finish with exit code zero. Start their consumers with `--wait`; when running the
initialization jobs directly, use `docker compose up mc` without `--wait`.

## Ubuntu host proxy

Both Ubuntu services share proxy environment variables pointing to
`http://host.docker.internal:7897` by default. On Docker Desktop for macOS,
[`host.docker.internal`](https://docs.docker.com/reference/cli/docker/container/run/#add-entries-to-container-hosts-file---add-host)
resolves to the host; `127.0.0.1` inside a container refers to that container.
The default port matches the local Clash Verge mixed HTTP/SOCKS proxy.
Keep Clash running with **Allow LAN** enabled and a bind address that accepts
container connections (the local configuration currently uses `*`).

Override the proxy endpoint in `.env` if the host proxy uses another HTTP or mixed
port, for example:

```dotenv
UBUNTU_PROXY_URL=http://host.docker.internal:7890
```

Apply changes by recreating just the Ubuntu services:

```sh
docker compose up -d ubuntu22 ubuntu24
```

Lowercase and uppercase `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, and `NO_PROXY`
variables are set for tools that support them, such as `apt`, `curl`, and Git over
HTTP(S). HTTPS destinations also use the `http://` proxy URL through HTTP CONNECT.
Localhost and the Compose service names bypass the proxy. Set `UBUNTU_NO_PROXY`
in `.env` to override the comma-separated bypass list, or `UBUNTU_PROXY_URL=` to
disable these proxy variables. Programs that ignore proxy environment variables,
including SSH, need their own proxy configuration. Docker image pulls use
[Docker Desktop's proxy settings](https://docs.docker.com/desktop/features/networking/#using-docker-desktop-with-a-proxy).

## Images and storage

Versions checked on 2026-10-06. Fixed release tags keep database major versions
predictable; mutable release channels are pinned to the manifests tested locally.

| Service               | Image version                                           | Persistent directory                                                  | Local port      |
| --------------------- | ------------------------------------------------------- | --------------------------------------------------------------------- | --------------- |
| MySQL                 | `mysql:26.7.0`                                          | `volumes/mysql/data`                                                  | 3306            |
| Redis                 | `redis:8.10.2-alpine`                                   | `volumes/redis/data`                                                  | 6379            |
| MongoDB               | `mongo:9.0.2`                                           | `volumes/mongo/data`, `volumes/mongo/configdb`                        | 27017           |
| PostgreSQL            | `postgres:18.6-alpine`                                  | `volumes/postgres/data/18/docker`                                     | 5432            |
| MinIO                 | Chainguard MinIO, digest pinned                         | `volumes/minio/data`                                                  | 9000, 9001      |
| MinIO client          | Chainguard MinIO client, digest pinned                  | None                                                                  | None            |
| Langfuse web / worker | Official `:4` manifests reporting 4.53.0, digest pinned | Shared PostgreSQL, ClickHouse, Redis, MinIO                           | 3100 / internal |
| ClickHouse            | `clickhouse/clickhouse-server:26.9.11.2-alpine`         | `volumes/clickhouse/data`                                             | Internal        |
| etcd                  | `quay.io/coreos/etcd:v3.7.2`                            | `volumes/etcd/data`                                                   | Internal        |
| Milvus                | `milvusdb/milvus:v3.0.2`                                | `volumes/milvus/data`, shared etcd and MinIO                          | 19530, 9091     |
| Kafka                 | `apache/kafka:4.3.1`                                    | `volumes/kafka/data`, `volumes/kafka/config`, `volumes/kafka/secrets` | 9092            |
| Nginx                 | `nginx:1.31.6-alpine-slim`                              | `volumes/nginx/html` (website files)                                  | 8080            |
| Prometheus            | `prom/prometheus:v3.15.0`                               | `volumes/prometheus/data`                                             | 9090            |
| Grafana               | `grafana/grafana:13.2.3`                                | `volumes/grafana/data`                                                | 3000            |
| Ubuntu 22 (`ubuntu22`) | `ubuntu:22.04`                                          | `volumes/ubuntu22/workspace`                                          | None            |
| Ubuntu 24 (`ubuntu24`) | `ubuntu:24.04`                                          | `volumes/ubuntu24/workspace`                                          | None            |

The PostgreSQL mount targets `/var/lib/postgresql`, as required by the PostgreSQL
18+ image layout. Old database directories are not migrated. MinIO initializes the
valid S3 bucket names `langfuse` and `milvus`; bucket names need at least three characters.
Grafana provisions a Prometheus datasource at `http://prometheus:9090`.
Kafka host clients use `localhost:9092`; Compose clients use `kafka:19092`.
Milvus has a 2 GiB memory limit for this local setup.

Redis uses the current
[Docker official Alpine image](https://github.com/docker-library/official-images/blob/master/library/redis).
Its server options are passed directly to `redis-server`, including AOF persistence, warning logging and the
`noeviction` policy needed by Langfuse's queues. RedisInsight is no longer bundled,
and port 8001 is no longer published.

One upstream lifecycle exception: [MinIO Community is archived](https://github.com/minio/minio).
The maintained Chainguard builds follow the storage choice in
[Langfuse's official Compose](https://github.com/langfuse/langfuse/blob/main/docker-compose.yml).
These builds are from Chainguard, not MinIO upstream.

Other version sources: [Docker official image definitions](https://github.com/docker-library/official-images/tree/master/library),
[Kafka](https://kafka.apache.org/blog/), [Milvus](https://github.com/milvus-io/milvus/releases),
[etcd](https://github.com/etcd-io/etcd/releases), [ClickHouse](https://github.com/ClickHouse/ClickHouse/releases),
[Prometheus](https://github.com/prometheus/prometheus/releases),
[Grafana](https://github.com/grafana/grafana/releases). The pulled Langfuse image labels
confirm the web and worker versions and source revision match.

## Disk and logs

- Every container uses Docker's `local` log driver: two 5 MB files, with compression.
  This is a rotation limit per container, not a limit on database storage.
- Application logs use warning/error where supported. MongoDB uses `--quiet` and
  disables diagnostic data collection because it has no global warn severity filter.
  Startup, database migration, and some mandatory server messages can still appear.
- Redis logs go to the container stream. ClickHouse file logging is disabled. Kafka
  application and GC logs go to the container stream. ClickHouse disables routine
  diagnostic system log
  tables, but retains `system.query_log` for one day because Langfuse v4 reads it;
  its file log directory is a 16 MB tmpfs.
- MySQL binary logging is disabled for local development; replication and binlog
  based point-in-time recovery require enabling it explicitly.
- Kafka deletes messages after 24 hours or approximately 256 MiB **per partition**;
  segments are 64 MiB. Retention is asynchronous, not a whole-broker disk quota.
- Prometheus retains metrics for three days or 512 MiB of stored blocks. WAL and
  current head data need extra space beyond that setting.
- etcd limits its backend quota to 512 MiB, retains two WAL files and two snapshots,
  and compacts revisions automatically. These limits do not periodically defragment
  the backend file; use `etcdctl defrag` when necessary.
- Grafana does not automatically preinstall optional plugins. User database and
  object storage records are kept until explicitly deleted.

Inspect usage with `docker system df` and `du -sh volumes/*`. Stop unused services
with `docker compose stop <service>`. Pull only selected services to avoid downloading
every image. To update a pinned manifest, inspect the upstream image, update its digest
in `x-images`, then validate and recreate that service.

`docker compose down` removes this project's containers and network; bind-mounted
files under `volumes/` remain on disk even with `down -v`.

## Validation

Validated on the local ARM64 Docker engine: all 14 long-running services became
healthy, both initialization jobs completed successfully, and Ubuntu's workspace
was writable. Read/write checks covered MySQL, PostgreSQL, MongoDB, Redis
modules, Kafka, etcd, MinIO and Milvus, including reads after container recreation.
An authenticated Langfuse OTLP trace reached ClickHouse through the worker.
Container inspection confirmed the bind mounts, loopback ports and log rotation.
Database persistence probes, test containers and images pulled only for validation
were cleaned up; the originally running MySQL and Redis services remain running.
The small Langfuse smoke trace remains available as verification data.

After switching to official Redis 8.10.2, string, JSON, search, Bloom and time-series
reads were checked again after container recreation. Inspection confirmed AOF
persistence, warning logging, `noeviction`, the single `/data` bind mount and only
port 6379. The replaced Stack image and unused RedisInsight data were removed.
The BullMQ client from the pinned Langfuse worker image successfully created and
processed a temporary job on official Redis; its test queue was then deleted.
