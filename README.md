# Java gRPC service for Invicti

Java 21 gRPC service with a MySQL-backed task API and a responsive task-manager web UI. It uses the existing MySQL container and a separate `grpc_invicti` database inside it; Compose does not start or publish another MySQL server. Envoy provides one TLS endpoint for the UI, native gRPC, HTTP/JSON transcoding, and the OpenAPI document.

## Architecture

```text
Browser UI ───────────────── HTTPS :8100 ─────────┐
Invicti gRPC scan (.proto) ─ HTTPS/HTTP2 :8100 ───┼─> Envoy ── gRPC ──> Java service ──> existing MySQL :3306
Invicti REST scan (OpenAPI) ─ HTTPS/JSON :8100 ────┘
```

Only Envoy is published. The host port defaults to **8100**, which was not occupied by any container in the `docker ps` listing you supplied. The UI, HTTP/JSON API, OpenAPI definition, and native gRPC all use that port. MySQL stays private on the existing Docker network.

## VPS setup

Requirements: Docker Compose v2, the existing `rml-demo-mysql` container, and a trusted TLS certificate for `grpc.rml.my.id`.

1. Find the Docker network attached to your MySQL container:

   ```sh
   docker inspect -f '{{range $name, $_ := .NetworkSettings.Networks}}{{println $name}}{{end}}' rml-demo-mysql
   ```

2. Copy `.env.example` to `.env`. Set `MYSQL_DOCKER_NETWORK` to the network name from step 1. Keep `DB_HOST=rml-demo-mysql` and `DB_PORT=3306`: containers on a shared Docker network use the MySQL container port, not the VPS host port. Also replace `DB_PASSWORD` and `JWT_SECRET`; generate a JWT secret with `openssl rand -hex 32`.

3. Put the same unique database password in both placeholders in `scripts/provision-mysql.sql`, then provision the dedicated database and user on the existing server:

   ```sh
   ./scripts/provision-existing-mysql.sh
   ```

   This runs SQL against the existing `rml-demo-mysql` container. It creates database `grpc_invicti` and the scoped user `grpc_invicti_app`; it does not create another MySQL server. The Java service creates its tables when it starts. The app user is not MySQL root.

4. Install the TLS certificate and private key as `deploy/certs/tls.crt` and `deploy/certs/tls.key`. The certificate must include `grpc.rml.my.id`.

5. Keep `PUBLIC_PORT=8100`, allow inbound TCP 8100 in the VPS firewall, then start the project:

   ```sh
   docker compose up -d --build
   docker compose ps
   docker compose logs -f envoy grpc-server
   ```

Open `https://grpc.rml.my.id:8100`. If this host port is already occupied by a process not shown in `docker ps`, choose another free port in `.env`.

### MySQL port note

The `docker ps` output you pasted shows `rml-demo-mysql` with internal `3306/tcp`, but does not show a published host mapping such as `0.0.0.0:3006->3306/tcp`. This project uses the MySQL container directly over its shared Docker network at `rml-demo-mysql:3306`, so there is no need to publish MySQL port 3006 to the internet. If the network-name command returns more than one network, use the network shared with the services that already connect to `rml-demo-mysql`.

## DAST test account

Demo-user creation is disabled by default. For an isolated DAST environment only, set `SEED_DEMO_USER=true`, choose `DEMO_USERNAME`, and set a unique `DEMO_PASSWORD` of at least 16 characters in `.env`. The service hashes the password with PBKDF2 and inserts the account only if it does not already exist. Use those values in the UI and Invicti authentication profile. Do not use `test/test` or enable the demo account on a public production service.

## Endpoints

- Web UI: `https://grpc.rml.my.id:8100`
- Native gRPC over TLS: `grpc.rml.my.id:8100`
- HTTP/JSON: `https://grpc.rml.my.id:8100/api/v1/...`
- OpenAPI file: `openapi/target.openapi.json`
- gRPC proto: `backend/src/main/proto/invicti/v1/invicti.proto`

For direct gRPC scanning, import the `.proto` file in Invicti and target the TLS endpoint. For HTTP/JSON scanning, import `openapi/target.openapi.json` and configure the login request and bearer token. Invicti editions have different native gRPC settings; Invicti Standard requires HTTP/2 and its HTTP client option enabled. HTTP/JSON scans do not cover every native gRPC transport behavior.

The API routes call the same Java gRPC implementation as native gRPC traffic. Passwords are hashed with PBKDF2, SQL inputs use prepared statements, JWT and database secrets have no built-in fallback in the server, and task access is owner-scoped. Envoy and Java log requests/RPCs; inspect with `docker compose logs -f envoy grpc-server`.

## OpenAPI generation

`openapi/target.openapi.json` is included as the import-ready definition. Regenerate it from the annotated proto with Docker and Buf:

```sh
./scripts/generate-openapi.sh
```

The remote OpenAPI plugin in `buf.gen.yaml` needs network access on first use.

## Security status

This version avoids starting another MySQL service and removes default app/JWT credentials. I would still treat it as a DAST lab starter, not fully production-hardened: add login throttling/lockout, review the custom JWT code, keep MySQL restricted to the Docker network, use trusted TLS, and scan with dedicated low-privilege accounts. The included self-signed certificate generator is for local development only.

## Deliberately vulnerable XSS demonstrations

This workspace now includes two intentional browser-side XSS flaws for authorized DAST practice. Stored XSS can be demonstrated by creating a task whose title or description contains HTML/JavaScript; it is saved to MySQL and interpreted when the task list is rendered. Reflected XSS uses the `q` query parameter (for example, `/?q=...`), which is inserted into the search feedback as HTML. Test only in an isolated environment with accounts and data you control. These flaws are unsafe for production; restore the backup or escape/insert user content as text before deployment.

The pre-change project snapshot is `grpc-invicti-java-before-xss-20261006.tar.gz`.

See [XSS-TESTING.md](XSS-TESTING.md) for setup, payloads, Invicti guidance, and restore steps. The frontend CSP allows inline scripts in this lab so the proof-of-concept event handlers can execute; remove that exception before production.

## Scope

No existing Java source or `.proto` files were present in the workspace, so this is a new starter project. It includes login and owner-scoped task operations. Streaming RPCs, gRPC reflection, and account administration are not included yet.
