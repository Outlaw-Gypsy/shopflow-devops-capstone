# ShopFlow — Cloud-Native Application Deployment

## 1. Project Overview

ShopFlow is a sample e-commerce application used to demonstrate the containerization and progressive deployment of a multi-service application. This repository currently provides a Docker-based local development environment consisting of a frontend, backend API, PostgreSQL database, and Redis service.

The project is being developed incrementally toward a broader cloud-native DevOps implementation. This README describes the functionality currently implemented and verified; additional CI/CD, Kubernetes, observability, security, and GitOps capabilities will be documented as they are introduced and tested.

## 2. Current Architecture

The application runs as four services orchestrated by Docker Compose.

| Component | Technology | Responsibility |
|---|---|---|
| Frontend | HTML, Nginx | Serves the web interface and proxies API requests |
| Backend | Python, Flask, Gunicorn | Provides application endpoints and business logic |
| Database | PostgreSQL | Stores application data persistently |
| Cache / data service | Redis | Provides the Redis service required by the backend |

### Request flow

1. A user accesses the application at `http://localhost:8080`.
2. Nginx serves the frontend page.
3. Requests to `/api/` are proxied to the Flask backend on port `5000`.
4. The backend communicates with PostgreSQL for persistent data and Redis for its Redis-dependent functionality.
5. Docker Compose manages service startup, dependency health conditions, and restart policies.

```text
Browser
   |
   | http://localhost:8080
   v
Frontend (Nginx)
   |
   | /api/* requests
   v
Backend (Flask + Gunicorn)
   |                  |
   v                  v
PostgreSQL           Redis
```

The frontend is the exposed entry point in the local Compose configuration. The backend, PostgreSQL, and Redis ports are not published directly to the host.

## 3. Repository Structure

```text
shopflow-devops-capstone/
├── backend/
│   ├── .dockerignore
│   ├── Dockerfile
│   ├── app.py
│   ├── requirements.txt
│   └── test_smoke.py
├── frontend/
│   ├── .dockerignore
│   ├── Dockerfile
│   ├── index.html
│   └── nginx.conf
├── .dockerignore
├── .gitignore
├── compose.yaml
└── README.md
```

## 4. Prerequisites

Install the following tools before starting the application:

- Docker Desktop, with the Docker engine running
- Docker Compose, available through `docker compose`
- Git
- Python 3.12 or a compatible Python environment for running the local smoke tests

The application containers use their own runtime environments, so Python does not need to be installed locally just to start the Compose stack.

## 5. Run the Application Locally

### Step 1: Clone the repository

```bash
git clone https://github.com/Outlaw-Gypsy/shopflow-devops-capstone.git
cd shopflow-devops-capstone
```

### Step 2: Configure the environment

Create a `.env` file in the repository root:

```dotenv
POSTGRES_PASSWORD=shopflow-local-dev-2026
```

This variable is consumed by Docker Compose when configuring PostgreSQL and the backend database connection.

**Security note:** This value is for local development only. Do not reuse it in production or commit the `.env` file to Git. Use a securely managed secret in a production deployment.

### Step 3: Build and start the services

```bash
docker compose up -d --build
```

Check the service status:

```bash
docker compose ps
```

PostgreSQL and Redis have health checks. The backend has a readiness-based health check and waits for PostgreSQL and Redis to become healthy before starting. The frontend depends on the backend being healthy.

### Step 4: Initialize the database

Run the application's database initialization command:

```bash
docker compose exec backend flask --app app init-db
```

To insert the sample products, run:

```bash
docker compose exec backend flask --app app seed
```

The seed command is intended to populate sample data for local verification. Check the API response to confirm the resulting records rather than assuming that repeated seed operations always produce the same database state.

### Step 5: Open the application

Visit:

`http://localhost:8080`

The frontend is served by Nginx on host port `8080`.

## 6. Verification and Health Checks

### Check application health

```bash
curl -i http://localhost:8080/health
```

Expected response: HTTP `200` with a healthy status.

### Check application readiness

```bash
curl -i http://localhost:8080/ready
```

The readiness endpoint checks the backend's PostgreSQL and Redis dependencies. A successful response indicates that the backend and these dependencies are ready.

### Check the products API

```bash
curl -i http://localhost:8080/api/products
```

After initializing and seeding the database, the response should contain the sample products.

### Check service status and logs

```bash
docker compose ps
docker compose logs --tail=100
docker compose logs backend
docker compose logs frontend
```

These commands help diagnose startup failures, unhealthy services, and application errors.

## 7. Automated Smoke Tests

The backend includes smoke tests in `backend/test_smoke.py`. The tests use an in-memory SQLite database for application testing and check the health endpoint and basic product creation/listing functionality.

To run them locally, create and activate a Python virtual environment, then install the backend dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r backend/requirements.txt
python -m pytest backend/test_smoke.py -v
```

The tests are separate from the PostgreSQL-backed Compose environment. Passing these smoke tests does not, by itself, prove that every integration or production scenario works.

## 8. Containerization and Implementation Decisions

### Backend image

The backend image uses `python:3.12-slim` as its base image. It installs the declared Python dependencies, runs the application through Gunicorn, and switches to a non-root application user.

Gunicorn provides a production-oriented WSGI server for running the Flask application inside the container.

### Frontend image and reverse proxy

The frontend image uses `nginx:alpine` to serve `index.html` and route API requests to the backend service.

The browser uses relative API paths such as `/api/products`. Nginx forwards those requests to the Compose service named `backend`, allowing the browser to use one application origin without publishing the backend port separately.

### PostgreSQL connectivity

The backend database URL uses the explicit SQLAlchemy driver scheme:

`postgresql+psycopg2://`

This matches the `psycopg2-binary` dependency declared in `backend/requirements.txt` and avoids relying on a different PostgreSQL driver.

### Persistent database storage

Compose defines a named volume, `postgres_data`, for PostgreSQL data. This allows database files to persist when containers are recreated.

Removing the containers does not normally remove this named volume. However, running `docker compose down -v` deletes the Compose-managed volume and its stored database data.

### Configuration and build context

Environment variables are used to configure the backend and database connection. Docker ignore files exclude unnecessary local files from image build contexts, including virtual environments, generated Python files, and local environment files.

## 9. Stop and Clean Up

To stop and remove the Compose containers and network while preserving the named database volume:

```bash
docker compose down
```

To also remove the Compose-managed database volume and its data:

```bash
docker compose down -v
```

Use the second command only when you intentionally want to delete the local database data.

## 10. Current Implementation Status

The following capabilities have been implemented and verified in the current local environment:

- Dockerfiles for the frontend and backend
- Docker Compose orchestration for frontend, backend, PostgreSQL, and Redis
- Nginx reverse proxy configuration for API and health endpoints
- PostgreSQL named-volume persistence
- PostgreSQL and Redis health checks
- Backend readiness health check and non-root container execution
- Database initialization and sample-data seeding
- Successful local responses from `/health`, `/ready`, and `/api/products`
- Backend smoke tests passing locally

These checks establish a working local containerized baseline. They do not imply that the remaining capstone requirements have already been implemented.

## 11. Planned Capstone Extensions

The next development stages will progressively add and verify:

- Git workflow evidence, including pull requests and conflict resolution
- Continuous integration and automated security checks
- Kubernetes deployments, services, configuration, probes, resource controls, and scaling
- Prometheus metrics, Grafana dashboards, alerting, and centralized logging
- Argo CD GitOps synchronization and drift detection
- A blue-green or canary deployment strategy with rollback evidence
- SLI/SLO definitions, error-budget calculations, incident procedures, and failure testing
- Supporting architecture, operational, and security documentation

Each capability will be marked as complete only after implementation and verification.

## 12. Repository

GitHub: https://github.com/Outlaw-Gypsy/shopflow-devops-capstone
