# FastAPI Traefik Setup and Audit

## The Project Run

I ran the FastAPI application locally.

Original Project: https://fastapi.tiangolo.com/
Original GitHub (only the framework, without an application): https://github.com/fastapi/fastapi
Original with template application (used by Liora): https://github.com/fastapi/full-stack-fastapi-template

Liora adapted repo with Traefik: https://github.com/DataScientest/fastapi-traefik-datascientest-project
Our cloned repo: https://github.com/fastapi-traefik-devops/fastapi-traefik-datascientest-project

### What is the Difference?

Liora changed **nothing**.
The DataScientest main branch is an exact copy of the official FastAPI template at commit: `d1df85e8098d72c0a43ff0da6dda8bb6662b3a44`
That commit was created in the official FastAPI repository on April 1, 2025. The identical commit exists as the head of the DataScientest repository. Because Git commit hashes include the parent, file tree and commit metadata, the same SHA means the repository contents at that point are identical.

The repository name contains: `fastapi-traefik-datascientest-project`
That gives the impression that DataScientest built a special Traefik version. In reality, Traefik was already a normal part of the upstream template.

So, as an idea — we can try to deploy the actual repository later.
Another idea — make some application on this template (ideas?).

### What Is Inside

Features that were already in the original template. The following were not introduced by Liora, although the course may present or use them as part of its DevOps exercises:

- FastAPI backend
- React, TypeScript and Vite frontend
- Chakra UI
- PostgreSQL
- SQLModel
- JWT authentication
- Password recovery by email
- Dockerfiles for frontend and backend
- Docker Compose
- Development overrides
- Traefik reverse proxy
- Automatic HTTPS through Let's Encrypt
- Production and staging deployment
- GitHub Actions
- Self-hosted GitHub Actions runners
- Backend tests with Pytest
- Frontend end-to-end tests with Playwright
- Adminer
- Generated frontend API client
- Copier-based project generation
- Pre-commit hooks
- Database health checks and migrations

### Outdated Differences from Current Upstream

This is a separate issue. The DataScientest repository is now a frozen April 2025 snapshot, while the official template continued developing.

As of August 5, 2026, the official repository has hundreds of additional commits. GitHub reports approximately 621 commits and 226 changed files between the copied commit and the current official branch.

Some notable later upstream changes include:

| April 2025 DataScientest snapshot                     | Current official template                            |
| ----------------------------------------------------- | ---------------------------------------------------- |
| Chakra UI                                             | Tailwind CSS and shadcn/ui                           |
| Separate frontend container                           | Frontend built into the backend image                |
| Frontend and backend on different development origins | Frontend can be served by FastAPI on the same domain |
| `docker-compose.yml`                                  | `compose.yml`                                        |
| `docker-compose.override.yml`                         | `compose.override.yml`                               |
| `docker-compose.traefik.yml`                          | `compose.traefik.yml`                                |
| Older Node/npm-oriented structure                     | Bun files and newer workspace tooling                |
| Approximately 789 commits                             | Approximately 1,410 commits                          |

The current official repository has a materially different structure and frontend stack.

## What Did I Do

To run it locally I did:
- `git clone git@github.com:fastapi-traefik-devops/fastapi-traefik-datascientest-project.git`
- `docker compose build` — to build the app. It uses the build rules from `docker-compose.yml`.
- `docker compose up` — to run all the containers.

I had a web server running on my PC on port 80, so it wouldn't run with `docker compose up` because of the port conflict. I stopped my web server:

```sh
sudo systemctl stop nginx
```

Additionally I used the command:

- `docker ps` — to show all running containers

```sh
CONTAINER ID   IMAGE                    COMMAND                  CREATED             STATUS                       PORTS                                                                                  NAMES
9c2d146362af   backend:latest           "fastapi run --reloa…"   About an hour ago   Up About an hour (healthy)   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp                                              fastapi-traefik-datascientest-project-backend-1
4ed9a280a7bf   adminer                  "entrypoint.sh docke…"   About an hour ago   Up About an hour             0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                                              fastapi-traefik-datascientest-project-adminer-1
ed074f17118b   traefik:3.0              "/entrypoint.sh --pr…"   About an hour ago   Up About an hour             0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:8090->8080/tcp, [::]:8090->8080/tcp         fastapi-traefik-datascientest-project-proxy-1
5f9f5b899e6e   frontend:latest          "/docker-entrypoint.…"   About an hour ago   Up About an hour             0.0.0.0:5173->80/tcp, [::]:5173->80/tcp                                                fastapi-traefik-datascientest-project-frontend-1
e001cb34e638   postgres:12              "docker-entrypoint.s…"   About an hour ago   Up About an hour (healthy)   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                              fastapi-traefik-datascientest-project-db-1
34f9ca8fdffa   schickling/mailcatcher   "sh -c 'mailcatcher …"   About an hour ago   Up About an hour             0.0.0.0:1025->1025/tcp, :::1025->1025/tcp, 0.0.0.0:1080->1080/tcp, :::1080->1080/tcp   fastapi-traefik-datascientest-project-mailcatcher-1
vitovt@T1001:~/Desktop/Dev/fastapi-traefik-datascientest-project$
```

I see the following application containers:

- backend — something related to backend
- frontend — something related to frontend
- traefik — proxy server
- postgresql — database server
- adminer — SQL admin web GUI
- mailcatcher — simple testing SMTP server for developers

After `docker compose up` I can access the **main GUI of the template application**:

http://localhost:5173/ with default credentials from `env`:
- user: admin@example.com
- pass: changethis

and the **Adminer GUI** to get direct access to the database:

http://localhost:8080/ with default credentials from `env`:
- user: postgres
- pass: changethis
- host: localhost
- db: app

All default credentials are in the `.env` file in the root of the repository.

## docker-compose.yml Analysis

This `docker-compose.yml` deploys the complete production stack:

- **PostgreSQL 12** with persistent storage and a readiness health check.
- **Adminer** for database management, exposed through Traefik at `https://adminer.${DOMAIN}`.
- **Prestart job** that waits for PostgreSQL, then runs `backend/scripts/prestart.sh`, typically for migrations and initial admin creation.
- **FastAPI backend** built from `./backend`, started only after the database is healthy and prestart succeeds; exposed at `https://api.${DOMAIN}`.
- **React frontend** built from `./frontend` with the production API URL embedded; exposed at `https://dashboard.${DOMAIN}`.

Traefik handles routing, HTTP-to-HTTPS redirects, TLS certificates through Let's Encrypt, and access through the external `traefik-public` network.

Required configuration comes from `.env`; expressions such as `${VARIABLE?Variable not set}` make Compose fail immediately when mandatory variables are missing.

Startup order:

```text
PostgreSQL healthy
        ↓
prestart completes successfully
        ↓
backend starts

frontend and Adminer start independently
```

The file does **not start Traefik itself**. It assumes an existing Traefik instance, external `traefik-public` network, and `https-redirect` middleware.

## Docker Compose Commands Cheatsheet

A practical workflow for the most common developer/devops tasks.

### Build

```sh
# Build all images (first time or after changing Dockerfiles)
docker compose build

# Build without using the cache (full rebuild from scratch)
docker compose build --no-cache

# Build a single service
docker compose build backend
```

### Run / Start

```sh
# Build (if needed) and start all containers in the background
docker compose up -d

# Start existing containers without rebuilding
docker compose start

# Run in the foreground (logs stream to the terminal)
docker compose up

# Start only one service
docker compose up -d db
```

### Check Status

```sh
# List running containers
docker compose ps

# Show logs of all services (follow mode)
docker compose logs -f

# Show logs of one service
docker compose logs -f backend

# Show resource usage
docker stats
```

### Stop / Destroy

```sh
# Stop all containers (keep them, can be restarted)
docker compose stop

# Stop and remove containers, networks, and volumes created by up
docker compose down

# Also remove named volumes (WARNING: deletes database data)
docker compose down -v

# Also remove images used by services
docker compose down --rmi all
```

### Rebuild / Update

```sh
# Pull latest images, rebuild, and restart changed containers
docker compose up -d --build

# Rebuild a single service and restart it
docker compose up -d --build backend

# Pull new versions of images without recreating containers
docker compose pull
```

### Common Workflow Combinations

```sh
# Full clean rebuild (after changing Dockerfiles or dependencies)
docker compose down
docker compose build --no-cache
docker compose up -d

# Quick restart after code changes (if code is mounted as a volume)
docker compose restart backend

# Update a running stack to the latest images
docker compose pull
docker compose up -d
```

### Useful Extras

```sh
# Run a one-off command inside a running service container
docker compose exec backend bash

# Run a one-off command in a new container (e.g. migrations)
docker compose run --rm backend alembic upgrade head

# Check the effective merged configuration
docker compose config
```

### Options Used Above

| Short | Long | Description |
|---|---|---|
| `-d` | `--detach` | Run containers in the background (don't block the terminal) |
| `-f` | `--follow` | Follow log output as it grows (`logs` command) |
| `-v` | `--volumes` | Also remove named volumes (`down` command — deletes data) |
| —    | `--rm`     | Remove the container automatically after it exits (`run` command) |
| —    | `--rmi`    | Also remove images used by services (`down` command) |
| —    | `--build`  | Build images before starting containers (`up` command) |
| —    | `--no-cache` | Build images without using the layer cache (full rebuild) |
| —    | `--pull`   | Always pull the latest images before starting (`up` command) |

## Reference Information

### Dockerfile vs Docker Compose

The core difference is that a Dockerfile is used to build a single Docker image, whereas Docker Compose is used to run and manage multiple Docker containers at the same time. [1, 2]
Think of a Dockerfile as a recipe for a single dish, and Docker Compose as the entire dinner menu and table arrangement. They are not competitors; they work together. [2, 3, 4]

#### Quick Comparison

| Feature | Dockerfile | Docker Compose |
|---|---|---|
| Primary Purpose | Builds an environment (Image). | Runs an environment (Containers). |
| Scope | Focuses on a single component. | Focuses on the whole application system. |
| File Format | Plain text file named Dockerfile. | YAML file named docker-compose.yml. |
| Main Command | docker build . | docker compose up |

---

#### What is a Dockerfile?

A Dockerfile is a script of sequential instructions that tells Docker exactly how to construct your custom application image. It sets up the operating system, installs software packages, copies your code, and specifies the startup command. [2, 5, 6, 7]

Example:

```dockerfile
# 1. Start with a base environment
FROM node:20-alpine
# 2. Set the working folder inside the image
WORKDIR /app
# 3. Install required software dependencies
COPY package*.json ./
RUN npm install
# 4. Copy the application source code
COPY . .
# 5. Define the command that executes when launched
CMD ["node", "server.js"]
```

---

#### What is Docker Compose?

Docker Compose is a tool used to define, launch, and connect multi-container setups. Instead of typing long, complex terminal commands every time you want to connect your application to a database or a cache, you define all components, storage disks, and network links in a single file. [5, 7, 8, 9, 10]

Example:

```yaml
version: '3.8'
services:
  # Service A: Your web application
  web-app:
    build: .                 # Automatically builds using the local Dockerfile
    ports:
      - "3000:3000"          # Links network ports
    depends_on:
      - database             # Ensures the database starts up first

  # Service B: A standard database engine
  database:
    image: postgres:15       # Pulls a ready-made image from Docker Hub
    environment:
      POSTGRES_PASSWORD: secret_password
```

---

#### How They Work Together

In a typical development workflow, you will use both files simultaneously: [2, 3]

1. You write a Dockerfile to package your specific app code into a runnable image.
2. You write a docker-compose.yml file to outline how that application image should talk to a database container (like MySQL or Postgres).
3. You run `docker compose up` to build the app image, provision the database, connect them over an internal network, and start your entire local environment with one command. [2, 5, 11, 12]

---

### k8s vs k3d vs Minikube

Here is the direct comparison: Kubernetes (K8s) is the full, production-ready system, while Minikube and k3d are tools used to run Kubernetes test environments locally on your PC.

#### Main Differences at a Glance

- **K8s (Kubernetes):** The original, powerful orchestration tool built for massive production server clusters.
- **Minikube:** Creates a full-featured, local K8s environment inside a virtual machine or a Docker container.
- **k3d:** Runs extremely fast by launching the lightweight k3s distribution directly inside Docker containers. k3d is distributed as a **single binary file** that you just run.

#### Detailed Breakdown

##### 1. Kubernetes (K8s)

- **Purpose:** Running containerized applications in production clouds or data centers.
- **Architecture:** Consists of separate, dedicated control planes and worker nodes.
- **Resources:** High footprint, requiring multiple full-scale servers.
- **Complexity:** Very high setup, configuration, and maintenance overhead.

##### 2. Minikube

- **Purpose:** Local development and authentic testing of official K8s features.
- **Architecture:** Simulates a standard K8s cluster on a single machine.
- **Drivers:** Uses virtual machines (VirtualBox, KVM) or containers (Docker, Podman).
- **Pro:** Supports almost all official K8s add-ons out of the box.
- **Con:** Slow startup time and heavy RAM consumption.

##### 3. k3d

- **Purpose:** Blazing fast local development and testing multi-node setups.
- **Architecture:** Wraps resource-friendly k3s (built for IoT/Edge) inside Docker.
- **Speed:** Boots up completely within a few seconds.
- **Pro:** Extremely lightweight and ideal for CI/CD automation pipelines.
- **Con:** Some niche K8s features are stripped out or behave differently.

#### Core Structural Differences

| Feature | k3d | minikube |
|---|---|---|
| Underlying K8s | k3s (Lightweight, optimized binaries) | Upstream Kubernetes (Full vanilla distribution) |
| Runtime Architecture | Runs directly in Docker containers | Runs in a VM (VirtualBox/QEMU) or via Docker driver |
| Resource Usage | Very low (~512MB RAM minimum) | High (~2GB RAM minimum baseline) |
| Startup Speed | Fast (~10–30 seconds) | Slow (~1–3 minutes) |
| Database Engine | SQLite (Default) or etcd | etcd (Heavy production-grade database) |
| Multi-Node Support | Native and rapid creation of multiple nodes/clusters | Primarily single-node (multi-node is heavy/complex) |
| Built-in Addons | Minimal out-of-the-box system | Massive plugin ecosystem (Dashboard, Ingress, Istio) |

#### When to Choose k3d

- **Resource Limits:** Your laptop has limited RAM or CPU space.
- **Fast Cycles:** You need clusters to spin up, tear down, or reboot instantly.
- **Multi-Node Simulation:** You want to test how routing works across multi-node setups without melting your laptop.
- **CI/CD Pipelines:** You require a tool to spin up transient environments inside CI jobs quickly.

#### When to Choose minikube

- **Full Parity:** You need 100% exact upstream conformance matching production clouds.
- **No Docker Dependency:** You prefer a dedicated VM instance rather than pinning everything to your local Docker engine.
- **Turnkey Learning:** You want instant access to managed components like dashboards, metrics, and service meshes via simple commands like `minikube addons enable`.