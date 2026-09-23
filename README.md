# ci-container-monitoring-lab

A small, fully local DevOps stack that wires a single commit all the way to a
running, monitored service:

**commit → CI → container → deploy → dashboard**

The stack contains:

- **app/** — a Node.js + Express service exposing `/`, `/health`, `/metrics` and `/work`
- **Dockerfile** — builds the service image
- **docker-compose.yml** — runs the app, Prometheus and Grafana together
- **prometheus/** — Prometheus scrape configuration
- **grafana/** — provisioned datasource and a "Service Overview" dashboard
- **.github/workflows/ci.yml** — GitHub Actions pipeline (install, test, build, smoke test)

Everything runs locally and for free. No cloud account or billing is required.

## Prerequisites

- Docker + Docker Compose
- Node.js 18+ (only needed if you want to run tests outside a container)
- `git`, `curl`

## Run the stack

```bash
docker compose up -d          # build + start app, Prometheus, Grafana
docker compose ps             # show container status and health
docker compose logs -f app    # follow the application logs
```

Services once up:

| Service     | URL                      |
|-------------|--------------------------|
| App         | http://localhost:8080    |
| Prometheus  | http://localhost:9090    |
| Grafana     | http://localhost:3000    |

Grafana logs in anonymously (admin/admin also works). Open the **Service
Overview** dashboard under the **Lab** folder.

Tear down:

```bash
docker compose down
```

## Verify the service

```bash
curl localhost:8080/health     # -> {"status":"ok"}
curl localhost:8080/           # -> service metadata
curl localhost:8080/metrics    # -> Prometheus metrics
```

Check the Prometheus scrape target:

```
http://localhost:9090/targets   # the app target should be UP
```

## Generate sample traffic

```bash
./scripts/generate-traffic.sh                       # defaults to localhost:8080
./scripts/generate-traffic.sh http://localhost:8080 500
```

Then watch the panels in the Grafana **Service Overview** dashboard update.

## Run the tests locally

```bash
cd app
npm install
npm test
```

## GitHub Actions

The workflow in `.github/workflows/ci.yml` runs automatically on every push and
pull request. It installs dependencies, runs the unit tests, builds the Docker
image and smoke-tests the running container. Watch it under the **Actions** tab
of your fork, or trigger it by pushing a commit / opening a PR.

## Rollback

Releases are tagged in Git. You can inspect history with:

```bash
git log --oneline
git tag
```

To roll a bad release back to a previous version, either revert the offending
commit and redeploy, or rebuild from a previous tag:

```bash
git revert <commit>          # revert a bad release
docker compose up -d --build # redeploy the reverted version
```

## Cloud Mapping (Local to GCP / Production)

The local pipeline maps directly to production cloud architecture on Google Cloud Platform (GCP):

| Stage | Local Flow | GCP Cloud Equivalent | Implementation & Architecture |
|---|---|---|---|
| **1. Commit & CI** | Git push triggers GitHub Actions workflow (`ci.yml`), running Node.js unit tests (`npm test`). | **GitHub Actions + GCP Workload Identity Federation** | On push, GitHub Actions authenticates with GCP, runs unit tests, and builds the container image. |
| **2. Container Registry** | Docker image built locally (`ci-container-monitoring-lab:ci`). | **Artifact Registry** | Image is built, tagged with Git commit SHA (`gcr.io/$PROJECT_ID/app:$COMMIT_SHA`), and pushed to GCP Artifact Registry. |
| **3. Container Deployment** | `docker compose up -d` deploys service container with environment variables (`MAINTENANCE_MODE=off`). | **Google Cloud Run** | Cloud Run deploys the container from Artifact Registry, managing scaling, TLS termination, and health check probes targeting `/health`. |
| **4. Monitoring & Alerting** | Prometheus scrapes `/metrics` endpoint; Grafana visualizes request rate and latency. | **Cloud Monitoring + Managed Service for Prometheus** | Cloud Monitoring ingests container logs and Prometheus metrics endpoints via Google Cloud Managed Service for Prometheus, driving Cloud Monitoring dashboards & uptime alerts. |
| **5. Instant Rollback** | `git revert <commit>` & `docker compose up -d --build` rebuilds and redeploys prior working state. | **Cloud Run Revision Traffic Splitting** | Cloud Run maintains immutable revisions per deployment. A bad release is rolled back instantly in seconds via `gcloud run services update-traffic app --to-revisions=PREVIOUS_REVISION=100`. |

