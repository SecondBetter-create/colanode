<div align="center">
<img alt="Colanode cover" src="assets/images/colanode-github-cover.jpg">
<p></p>
<a target="_blank" href="https://opensource.org/licenses/Apache-2.0" style="background:none">
    <img src="https://img.shields.io/badge/Licene-Apache_2.0-blue" style="height: 22px;" />
</a>
<a target="_blank" href="https://discord.gg/ZsnDwW3289" style="background:none">
    <img alt="" src="https://img.shields.io/badge/Discord-Colanode-%235865F2" style="height: 22px;" />
</a>
<a href="https://x.com/colanode" target="_blank">
  <img alt="" src="https://img.shields.io/twitter/follow/colanode.svg?style=social&label=Follow" style="height: 22px;" />
</a>
</div>

# Colanode

### Open-source & local-first collaboration workspace that you can self-host

Colanode is an all-in-one platform for easy collaboration, built to prioritize your data privacy and control. Designed with a **local-first** approach, it helps teams communicate, organize, and manage projects—whether online or offline. With Colanode, you get the flexibility of modern collaboration tools, plus the peace of mind that comes from owning your data.

### What can you do with Colanode?

- **Real-Time Chat:** Stay connected with instant messaging for teams and individuals.
- **Rich Text Pages:** Create documents, wikis, and notes using an intuitive editor, similar to Notion.
- **Customizable Databases:** Organize information with structured data, custom fields and dynamic views (table, kanban, calendar).
- **File Management:** Store, share, and manage files effortlessly within secure workspaces.

Built for both individuals and teams, Colanode adapts to your needs, whether you're running a small project, managing a team, or collaborating across an entire organization. With its self-hosted model, you retain full control over your data while enjoying a polished, feature-rich experience.

![Colanode preview](assets/images/colanode-desktop-preview.gif)

## How it works

Colanode includes a client app (web or desktop) and a self-hosted server. You can connect to multiple servers with a single app, each containing one or more **workspaces** for different teams or projects. After logging in, you pick a workspace to start collaborating—sending messages, editing pages, or updating database records.

### Local-first workflow

All changes you make are saved to a local SQLite database first and then synced to the server. A background process handles this synchronization so you can keep working even if your computer or the server goes offline. Data reads also happen locally, ensuring immediate access to any content you have permissions to view.

### Concurrent edits

Colanode relies on **Conflict-free Replicated Data Types (CRDTs)** - powered by [Yjs](https://docs.yjs.dev/) - to allow real-time collaboration on entries like pages or database records. This means multiple people can edit at the same time, and the system gracefully merges everyone's updates. Deletions are also tracked as specialized transactions. Messages and file operations don't support concurrent edits and use simpler database tables.

## Get started for free

The easiest way to start using Colanode is through our **web app**, accessible instantly at [app.colanode.com](https://app.colanode.com). Simply log in to get started immediately, without any installation. _Please note, the web app is currently in early preview and under testing; you may encounter bugs or compatibility issues in certain browsers._

For optimal performance, you can install our **desktop app**, available from our [downloads page](https://colanode.com/downloads). Both the web and desktop apps allow you to connect to any of our free beta cloud servers:

- **Colanode Cloud (EU)** – hosted in Europe.
- **Colanode Cloud (US)** – hosted in the United States.

Both cloud servers are currently available in beta and free to use; pricing details will be announced soon.

### Self-host

If you prefer to host your own Colanode server, check out the [`hosting/`](hosting/) folder which contains the Docker Compose file and deployment configurations. For Kubernetes deployments, see the [`hosting/kubernetes/`](hosting/kubernetes/) folder which includes Helm charts and additional documentation. Here's what you need to run Colanode yourself:

- **Postgres** with the **pgvector** extension.
- **Redis** (any Redis-compatible service will work, e.g., Valkey).
- **Storage backend** for user files. Colanode defaults to local filesystem storage, but you can switch to **S3-compatible**, **Google Cloud Storage**, or **Azure Blob Storage** backends by setting `STORAGE_TYPE`.
- **Colanode server API**, provided as a Docker image.

#### Configuration model

- The server image now ships with a full `config.json`, so most defaults are ready to go without touching env vars.
- The config file is the single source of truth. Use `env://VAR_NAME` to pull sensitive values from env vars, or `file://path/to/secret.pem` to inline the contents of a mounted file (append `?` to make either optional). Only `POSTGRES_URL` and `REDIS_URL` are required out of the box.
- To customize settings:
  1. Copy `apps/server/config.json`, edit it, and mount/bind it when using Docker Compose (see `hosting/docker/docker-compose.yaml`).
  2. For Helm, enable `colanode.configFile.enabled` and pass your file via `--set-file colanode.configFile.data=./config.json` (details in [`hosting/kubernetes/README.md`](hosting/kubernetes/README.md)).
  3. Keep secrets as env vars so you don't have to bake them into JSON; the loader resolves `env://` pointers at runtime.

Environment variables no longer override regular config fields—only values explicitly tagged with `env://` are read from the environment. Refer to [`hosting/docker/docker-compose.yaml`](hosting/docker/docker-compose.yaml) and [`hosting/kubernetes/README.md`](hosting/kubernetes/README.md) for mounting instructions and the handful of required secrets.

### Deploying to Railway

Railway does not run this Docker Compose file directly. Create the Compose services as separate Railway services/resources:

| Compose service | Railway setup | Public |
| --- | --- | --- |
| `postgres` | Use Railway's pgvector template (or deploy `pgvector/pgvector:pg17` with a volume mounted at `/var/lib/postgresql/data`). Configure the database credentials and database name as service variables. | No |
| `valkey` | Deploy `valkey/valkey:8.1`, set a generated alphanumeric `REDIS_PASSWORD`, and use start command `valkey-server --requirepass "$REDIS_PASSWORD"`. | No |
| `server` | Repository source, root directory `/`, Dockerfile path `apps/server/Dockerfile`. | Yes |
| `web` | Repository source, root directory `/`, Dockerfile path `apps/web/Dockerfile`. | Yes |
| `minio` (optional `s3` profile) | Deploy `minio/minio:RELEASE.2025-04-08T15-41-24Z`, attach a volume at `/data`, and use start command `minio server /data --address ":9000" --console-address ":9001"`. Set `MINIO_ROOT_USER` and a generated `MINIO_ROOT_PASSWORD`. | No |

Keep the repository root as the root directory for both application services; their Dockerfiles build shared workspace packages from the repository root. Add persistent volumes for Postgres, MinIO when enabled, and `server` mounted at `/data`. Do not expose database, cache, or MinIO ports publicly.

Expose the `web` service and `server` service on separate Railway domains. Colanode clients connect directly to the API and its WebSocket endpoint, so the server's domain must be public; use `https://<server-domain>/config` as the server URL. Railway supplies `PORT` to the app services, and both containers listen on it.

On `server`, set `CONFIG=/app/apps/server/config.railway.json`, `POSTGRES_URL` to the pgvector service's internal database URL, `REDIS_URL` to `redis://:<password>@<valkey-private-domain>:6379/0`, `WEB_DOMAIN` to the web service's public domain without a scheme, and `WEB_ORIGIN` to its origin including `https://`. Keep the Valkey password alphanumeric so it can be used in the URL without escaping.

The default Railway server config stores uploads on the `/data` volume. To use the optional MinIO service instead, set `CONFIG=/app/apps/server/config.railway.s3.json` on `server`, create a `colanode` bucket in MinIO, and set `S3_ENDPOINT` to `http://<minio-private-domain>:9000`, `S3_ACCESS_KEY` / `S3_SECRET_KEY` to the MinIO credentials, `S3_BUCKET=colanode`, and `S3_REGION=us-east-1`.

The Compose file's Mailpit SMTP service is commented out and is for local email testing; it is not part of the default Railway setup.

Desktop (Electron) and mobile (Expo/native) are client apps, not Railway web services.

### Deploying to Vercel

The root [`vercel.json`](vercel.json) deploys the Vite web app at `/` and the Fastify server at `/api/*`. Desktop and mobile are native clients, so they are not Vercel services. The server receives the original request path; [`apps/server/config.vercel.json`](apps/server/config.vercel.json) sets `pathPrefix` to `api`, including for `/api/config`.

Configure these environment variables on the `server` service in Vercel. Set `CONFIG=./config.vercel.json`; the file is included with the server function.

| Variable | Value |
| --- | --- |
| `POSTGRES_URL` | PostgreSQL connection URL for a database with the `vector` extension |
| `REDIS_URL` | Redis/Valkey connection URL |
| `WEB_DOMAIN` | Public web domain without a scheme, such as `colanode.example.com` |
| `WEB_ORIGIN` | Public web origin including the scheme, such as `https://colanode.example.com` |
| `S3_ENDPOINT` | S3-compatible storage endpoint |
| `S3_ACCESS_KEY` / `S3_SECRET_KEY` | S3 credentials |
| `S3_BUCKET` | Bucket for uploaded files |
| `S3_REGION` | S3 region |

Set the web domain and origin to match the Vercel domain for each deployment environment. The database, Redis, and object storage must be provisioned separately; Vercel service bindings are not needed because the browser connects directly to the configured server URL. Add a server in Colanode using `https://<deployment-domain>/api/config`.

The server uses WebSockets and starts queue/event consumers from its entrypoint. Vercel's Fastify deployment runs as a Function, and WebSocket support is currently in public beta; verify those runtime constraints and function limits for your Vercel plan before using this as a production deployment.

### Running locally

To run Colanode locally in development mode:

1. Clone the repository:

   ```bash
   git clone https://github.com/colanode/colanode.git
   cd colanode
   ```

2. Install dependencies at the project root:

   ```bash
   npm install
   ```

3. Start the apps you want to run locally:

   **Server**

   ```bash
   cd apps/server

   # Copy the environment variable template and adjust values as needed
   cp .env.example .env

   npm run dev
   ```

   To spin up the local dependencies (Postgres, Redis, and Mail server) with Docker Compose—using filesystem storage
   by default—run this from
   the project root:

   ```bash
   docker compose -f hosting/docker/docker-compose.yaml up -d
   ```

   When you prefer an S3-compatible backend locally, enable the optional MinIO service with the `s3` profile:

   ```bash
   docker compose -f hosting/docker/docker-compose.yaml --profile s3 up -d
   ```

   The compose file includes a `server` service. When you want to run the API locally with `npm run dev`, comment
   out (or override) that service so only the supporting services are started.

   **Web**

   ```bash
   cd apps/web
   npm run dev
   ```

   **Desktop**

   ```bash
   cd apps/desktop
   npm run dev
   ```

## Testing

Colanode includes tests for both server and web.

### Server tests

From `apps/server`:

```bash
npm run test
```

Server tests use Testcontainers for Postgres and Redis, so Docker must be running. See [`apps/server/README.md`](apps/server/README.md) for details.

### Web tests

From `apps/web`:

```bash
npm run test
```

## License

Colanode is released under the [Apache 2.0 License](LICENSE).
