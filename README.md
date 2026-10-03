# <a href="https://hangtiancheng.github.io/r/" target="_blank" rel="noopener noreferrer">Hang Tiancheng's Resume</a>

## Language Stats

**[hangtiancheng.github.io/hangtiancheng/language-stats.html](https://hangtiancheng.github.io/hangtiancheng/language-stats.html)**

## Yukino trace observability

Langfuse v4 uses the existing PostgreSQL, Redis, and MinIO services in the root
`docker-compose.yml`; ClickHouse, the Langfuse web app, and its worker are part
of the same Compose project.

```bash
cd ~/github/hangtiancheng
docker compose up -d --wait postgres redis minio mc clickhouse langfuse-web langfuse-worker
```

Open <http://localhost:3100> and sign in with the local credentials in `.env`.
The Yukino organization, project, and API keys are created on the first start.

Load the client variables before starting Yukino:

```bash
cd ~/github/yukino-code/apps/yukino
set -a
source ~/github/hangtiancheng/.env
set +a
pnpm dev
```

After an agent request completes, its trace appears in Langfuse under
**Tracing**. Useful operations:

```bash
cd ~/github/hangtiancheng
docker compose ps postgres redis minio clickhouse langfuse-web langfuse-worker
docker compose logs -f langfuse-web langfuse-worker
curl -fsS http://localhost:3100/api/public/health
```

`.env` contains local secrets and is ignored by Git. Copy `.env.example` when
setting up another machine.
