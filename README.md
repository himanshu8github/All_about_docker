# 🐳 Docker From Scratch: The Complete Step-by-Step Learning Guide

Welcome to the complete, zero-to-hero roadmap for mastering Docker. This guide is crafted to teach you not just *what commands to type*, but **how Docker works under the hood** in simple, human-friendly language.

---

## 🗺️ Roadmap Overview

| Step | Topic & Focus Area | Key Concepts |
| :--- | :--- | :--- |
| **Step 1** | **Docker Architecture & Core Mental Model** | Virtual Machines vs Containers, Linux Namespaces, cgroups |
| **Step 2** | **Images vs Containers (Under the Hood)** | Read-Only Layers, Copy-on-Write (CoW), Union File Systems |
| **Step 3** | **Disk Usage, Sizing & Under-the-Hood Inspection** | Virtual Size vs Writable Layer, exact byte size, finding OS/distro |
| **Step 4** | **Container Lifecycle & Port Binding Mechanics** | Host Port vs Container Port, Stop Signals, Restart Policies |
| **Step 5** | **Configuration & Environment Variables** | Inspecting configs, overriding Envs at runtime vs build-time |
| **Step 6** | **Deep Dive: Dockerfile Mastery** | `ENTRYPOINT` vs `CMD`, `COPY` vs `ADD`, `WORKDIR`, `HEALTHCHECK` |
| **Step 7** | **Image Optimization & Size Reduction** | Multi-stage builds, Alpine/Distroless, layer caching best practices |
| **Step 8** | **Persistent Data: Volumes & Bind Mounts** | Anonymous vs Named Volumes, Bind Mounts, Host storage |
| **Step 9** | **Docker Networking & Service Discovery** | Bridge, Host, DNS discovery, Round-Robin load balancing |
| **Step 10** | **Docker Registries & Image Distribution** | Docker Hub, Tags, Layer caching across repositories, Push/Pull |
| **Step 11** | **Docker Compose (v2) & Multi-Container Apps** | Services, dependencies, build/runtime overrides, Compose v2 |
| **Step 12** | **Full-Stack Project: MySQL + Backend + Frontend** | Production wiring, persistent DB, health checks, network isolation |
| **Step 13** | **Cleanup, Pruning & Maintenance** | Reclaiming disk space, removing stopped containers & dangling images |

---

## 🚀 Step 1: Docker Architecture & The Core Mental Model

### ❓ Question 1: What is a Container, and how is it fundamentally different from a Virtual Machine (VM)?

* **Virtual Machine (VM):**
  * Bundles a full guest Operating System (kernel + system utilities + libraries + your app).
  * Runs on top of a **Hypervisor** (Type 1 or Type 2) that virtualizes physical hardware (CPU, RAM, NIC).
  * **Trade-off:** Heavyweight (gigabytes in size), slow boot time (minutes), high memory footprint.
* **Docker Container:**
  * Does **NOT** run a separate OS kernel. It shares the **Host OS Kernel**.
  * It is simply an isolated Linux process running on your machine with restricted visibility and resources.
  * **Trade-off:** Lightweight (megabytes in size), instant boot (sub-second), native performance.

```
       +-------------------------+          +-------------------------+
       |   App 1   |    App 2    |          |   App 1   |    App 2    |
       +-----------+-------------+          +-----------+-------------+
       | Guest OS  |  Guest OS   |          |    Bins / Libraries     |
       +-----------+-------------+          +-------------------------+
       |       Hypervisor        |          |      Docker Engine      |
       +-------------------------+          +-------------------------+
       |   Host OS & Kernel      |          |    Host OS & Kernel     |
       +-------------------------+          +-------------------------+
       |    Physical Hardware    |          |    Physical Hardware    |
       +-------------------------+          +-------------------------+
         [ Virtual Machine Model ]               [ Container Model ]
```

### ❓ Question 2: What Linux features make containers possible?
Under the hood, Docker leverages two primary Linux kernel primitives:
1. **Namespaces (Isolation):** Determines *what a container can see*.
   * `pid` namespace: Container thinks its main process is PID 1.
   * `net` namespace: Dedicated virtual network interfaces, routing tables, and ports.
   * `mnt` namespace: Dedicated file system mount points.
   * `ipc` / `uts` / `user` namespaces: Isolate system communication, hostnames, and user IDs.
2. **Control Groups (cgroups) (Resource Metering):** Determines *how much a container can use*.
   * Enforces limits on CPU cores, RAM limits, disk I/O, and network bandwidth.

---

## 📦 Step 2: Images vs. Containers (Under the Hood)

### ❓ Question 3: What is a Docker Image, and what are Layers?
* An **Image** is a read-only, immutable blueprint or snapshot of an environment (code, runtime, dependencies, system libraries, configuration).
* An image is composed of a stack of **read-only layers**.
* Every directive in a `Dockerfile` (like `FROM`, `RUN`, `COPY`) creates a new layer cached by its cryptographic SHA-256 hash.
* If two images both use `ubuntu:22.04` as their base, Docker downloads that layer **only once** and shares it across all images on your machine.

### ❓ Question 4: How does a Container turn an Image into something writable? (UnionFS & CoW)
* When you run `docker run`, Docker mounts all the read-only image layers together into a single unified filesystem view using a **Union File System (UnionFS)** (e.g., `overlay2`).
* Docker adds a thin, **Read-Write Layer** (called the *Container Layer*) on the very top.
* **Copy-on-Write (CoW) Mechanism:**
  * When a container reads a file, it reads from the read-only image layers beneath.
  * If the container modifies an existing file, Docker copies that file *up* into the top writable layer, and modifies the copy. The original underlying image layer remains untouched.
  * When a container is deleted, only this thin top writable layer is destroyed.

```
+--------------------------------------------------------+
|   Top Writable Layer (Container Layer) - Reads & Writes|  <-- Destroyed on `docker rm`
+--------------------------------------------------------+
|   Layer 3: COPY ./app /app               (Read-Only)   |
+--------------------------------------------------------+
|   Layer 2: RUN apt-get install python3   (Read-Only)   |  <-- Shared across containers
+--------------------------------------------------------+
|   Layer 1: Base OS (e.g., debian:bookworm-slim) (R/O)  |
+--------------------------------------------------------+
```

---

## 📊 Step 3: Disk Usage, Sizing & Under-the-Hood Inspection

### ❓ Question 5: What is the difference between "Virtual Size" and "Writable Container Size"?
* **Virtual Size:** The total size of all underlying read-only layers + the container's writable layer.
* **Container Size:** The size of data currently written to the container's top writable layer (changes, logs, temporary files).

```bash
# View container sizes (shows both container layer size and virtual size)
docker ps -s
```

### ❓ Question 6: How do you inspect the exact image size in bytes and check overall disk usage?

```bash
# 1. Check overall Docker disk utilization
docker system df
docker system df -v

# 2. Inspect exact image size in raw bytes
docker image inspect <IMAGE_NAME_OR_ID> --format='{{.Size}} bytes'

# 3. View layer-by-layer breakdown of an image
docker history <IMAGE_NAME_OR_ID>
```

### ❓ Question 7: How can you determine which OS / Linux distribution is inside a container?

```bash
# Run a quick shell and inspect the OS release file
docker run --rm <IMAGE_NAME> cat /etc/os-release

# Alternative for minimal shells:
docker run --rm <IMAGE_NAME> uname -a
```

---

## ⚡ Step 4: Container Lifecycle & Port Binding Mechanics

### ❓ Question 8: What is Port Mapping (`-p host_port:container_port`), and what is the difference between Host and Port?
* **Host:** The physical or virtual machine running the Docker daemon (your laptop or cloud server). It has its own IP and port space.
* **Container:** An isolated network namespace with its own private virtual IP address (e.g., `172.17.0.2`) and internal ports.
* By default, outside traffic cannot reach the container's internal ports.
* **Port Binding (`-p 8080:80`):**
  * Tells Docker's network bridge / iptables / userland proxy: *"Forward incoming traffic on Host port 8080 to Container port 80"*.
  * Syntax: `-p <HOST_PORT>:<CONTAINER_PORT>`

```
              Host Machine (e.g. 192.168.1.50)
        +------------------------------------------+
Browser |  Port 8080                               |
------> |    │                                     |
        |    └───► Docker Proxy / iptables         |
        |                 │                        |
        |                 ▼ Forward                |
        |          Container (172.17.0.2)          |
        |          +--------------------+          |
        |          | Port 80 (Nginx/App)|          |
        |          +--------------------+          |
        +------------------------------------------+
```

### ❓ Question 9: What are Stop Signals and Restart Policies?
* **Stop Signals (`SIGTERM` vs `SIGKILL`):**
  * `docker stop`: Sends `SIGTERM` to PID 1 inside the container, granting a grace period (default 10s) to close database connections and finish tasks. If it doesn't terminate, it sends `SIGKILL`.
  * `docker kill`: Immediately sends `SIGKILL` without waiting.
* **Restart Policies (`--restart`):**
  * `no`: Never automatically restart (default).
  * `always`: Always restart if stopped or host reboots.
  * `on-failure[:max-retries]`: Restart only if process exits with a non-zero exit code.
  * `unless-stopped`: Restart unless manually stopped by the user.

```bash
# Example: Running an nginx container with auto-restart and port binding
docker run -d --name my-web -p 8080:80 --restart unless-stopped nginx:alpine
```

---

## 🔍 Step 5: Configuration & Environment Variables

### ❓ Question 10: How do you inspect container configuration and environment variables?

```bash
# 1. View all metadata (Network, Mounts, State, Config)
docker inspect <CONTAINER_NAME_OR_ID>

# 2. Extract only environment variables using format template
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' <CONTAINER_NAME>

# 3. Read environment variables from inside the live container
docker exec <CONTAINER_NAME> env
```

### ❓ Question 11: How do you pass or override Environment Variables?
* **At Container Runtime:**
  ```bash
  # Pass individual variables
  docker run -e DB_HOST=db.example.com -e DB_PORT=3306 my-app

  # Pass from an env file
  docker run --env-file .env.production my-app
  ```
* **At Build-Time vs Runtime:**
  * Build-time variables (`ARG`): Only available during `docker build`. Not baked into final running state unless explicitly passed to `ENV`.
  * Runtime environment variables (`ENV`): Available during both build-time and in the running container process.

---

## 🛠️ Step 6: Deep Dive: Dockerfile Mastery

### ❓ Question 12: What does each essential Dockerfile directive do?

| Directive | Purpose | Best Practice / Gotcha |
| :--- | :--- | :--- |
| `FROM` | Specifies base image | Use specific, trusted tags (e.g., `node:20-alpine`, not `latest`) |
| `WORKDIR` | Sets the working directory inside the container | Always use `WORKDIR /app` instead of `RUN cd /app` |
| `COPY` vs `ADD` | Copies files from host to image | Use `COPY`. `ADD` has extra behavior (extracts tarballs, fetches URLs) |
| `RUN` | Executes commands during image build | Chain commands with `&&` to reduce layers (`apt-get update && apt-get install -y ...`) |
| `CMD` vs `ENTRYPOINT` | Defines the container execution entrypoint | See Question 13 |
| `EXPOSE` | Documentation metadata indicating listening ports | Does **not** publish the port automatically! (Need `-p` or `-P`) |
| `LABEL` | Adds key-value metadata to the image | Useful for maintainer, version, description |
| `SHELL` | Overrides default shell | E.g., `SHELL ["/bin/bash", "-c"]` or PowerShell on Windows containers |
| `HEALTHCHECK` | Tells Docker how to test if container is healthy | `HEALTHCHECK --interval=30s CMD curl -f http://localhost/ || exit 1` |
| `STOPSIGNAL` | Customizes system call signal to terminate | E.g., `STOPSIGNAL SIGQUIT` (useful for Nginx graceful shutdown) |

### ❓ Question 13: What is the exact difference between `ENTRYPOINT` and `CMD`?
* `ENTRYPOINT`: The executable that **always runs**. It defines what the container *is*.
* `CMD`: Default arguments passed to the `ENTRYPOINT`. These can easily be overridden from the command line.

```dockerfile
# Example 1: Combined pattern (Industry Standard)
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```
* Running `docker run my-image` executes: `python app.py --port 8000`
* Running `docker run my-image --port 9000` overrides `CMD`: `python app.py --port 9000`

---

## 📉 Step 7: Image Optimization & Size Reduction

### ❓ Question 14: How do you shrink a 1GB Docker image down to 50MB?

1. **Use Minimal Base Images:** Switch from full Ubuntu/Debian (`~1GB`) to Alpine (`~5MB`) or Distroless (`~20MB`).
2. **Combine `RUN` Commands:** Each `RUN` creates a layer. Clean caches in the same command:
   ```dockerfile
   RUN apt-get update && apt-get install -y --no-install-recommends \
       curl \
       && rm -rf /var/lib/apt/lists/*
   ```
3. **Use `.dockerignore`:** Prevent copying `node_modules`, `.git`, temporary logs, or secrets into build context.
4. **Multi-Stage Builds:** Compile your code in a heavy build stage, then copy **only the final artifact** into a tiny runtime stage.

### 💡 Multi-Stage Build Example:

```dockerfile
# ── STAGE 1: Build stage (heavy) ──
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /bin/server .

# ── STAGE 2: Production runtime stage (tiny!) ──
FROM scratch
COPY --from=builder /bin/server /bin/server
EXPOSE 8080
ENTRYPOINT ["/bin/server"]
```
*(Result: Image drops from 800MB to ~15MB!)*

---

## 💾 Step 8: Persistent Data: Volumes & Bind Mounts

### ❓ Question 15: Where does container data go when a container is deleted?
* Because containers use a temporary writable layer, any data created inside is **lost** when the container is removed (`docker rm`).
* To persist data, Docker provides two main storage options:

| Feature | Docker Named Volume | Bind Mount |
| :--- | :--- | :--- |
| **Location** | Managed by Docker (`/var/lib/docker/volumes/` on Linux) | Any directory on your host machine (e.g. `./data`) |
| **Managed by** | Docker CLI (`docker volume ...`) | Host OS filesystem |
| **Best For** | Production databases, persistent caches | Local development (hot-reload code sharing) |
| **Portability** | High (independent of host directory structure) | Depends on host path existing |

```bash
# 1. Create and run with a Named Volume
docker volume create mysql_data
docker run -d -v mysql_data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=secret mysql:8

# 2. Run with a Bind Mount (ideal for live development hot-reload)
docker run -d -p 3000:3000 -v $(pwd)/src:/app/src my-dev-app
```

---

## 🌐 Step 9: Docker Networking & Service Discovery

### ❓ Question 16: What network drivers exist in Docker?
1. **Bridge (Default):** Creates a private virtual network on the host. Containers get private IPs (e.g., `172.17.x.x`).
2. **Host:** Container shares the host's network directly (no port mapping needed; maximum performance, less isolation).
3. **None:** Complete network isolation (no internet, no loopback interface).
4. **Overlay:** Multi-host networking across multiple Docker daemons (used in Docker Swarm & Kubernetes).

### ❓ Question 17: How does Docker Network DNS and Round-Robin Load Balancing work?
* In **custom user-defined bridge networks**, Docker runs an **embedded DNS server** at `127.0.0.11`.
* Containers can resolve each other simply by their **Container Name** or **Service Name** (no hardcoded IP addresses needed!).
* **DNS Round-Robin:** If multiple containers share the same network alias (e.g., two backend containers under alias `api`), Docker's DNS server rotates the IP returned for `api`, giving basic round-robin load balancing.

```bash
# Create custom network
docker network create my-network

# Run two containers connected to the same network
docker run -d --name db --network my-network mysql:8
docker run -d --name web --network my-network -p 8080:80 my-web-app

# Inside the 'web' container, pinging 'db' automatically resolves to MySQL's IP:
docker exec web ping db
```

---

## ☁️ Step 10: Docker Registries & Industry Best Practices

### ❓ Question 18: What is a Docker Registry (like Docker Hub), and how do tags work?
* An **Image Registry** is a central repository for storing and distributing versioned container images.
* **Tag anatomy:** `registry.domain.com/namespace/repository:tag`
  * Example: `docker.io/library/nginx:1.25-alpine`
* **Industry Best Practices:**
  * ❌ Avoid using `:latest` in production environments (it changes unpredictably).
  * ✅ Use explicit semantic versions (`:1.4.2`) or Git commit SHAs (`:sha-a1b2c3d`).
  * ✅ Leverage layer sharing: If 10 microservices share the same base layer, pulling them saves massive bandwidth and disk space.

---

## 🎼 Step 11: Multi-Container Orchestration with Docker Compose

### ❓ Question 19: What is Docker Compose (Compose v2)?
* `docker-compose` (v1, Python-based script) is deprecated.
* **Docker Compose v2** is rewritten in Go and integrated natively into the Docker CLI as `docker compose`.
* It defines and runs multi-container Docker applications using a single declarative YAML file (`docker-compose.yml`).

### ❓ Question 20: How do you configure building, dependencies, and environment overrides?

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      args:
        BUILD_ENV: production        # Build-time argument
    environment:
      - PORT=5000                   # Runtime environment variable
      - DB_HOST=database
    ports:
      - "5000:5000"
    depends_on:
      database:
        condition: service_healthy  # Wait until DB is actually healthy
    restart: always

  database:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: myapp
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 5s
      retries: 10

volumes:
  db_data:
```

---

## 🏗️ Step 12: Hands-On Project: Deploying MySQL + Backend + Frontend

Here is how you connect a full 3-tier architecture with health checks and data isolation:

```
[ User Browser ]
       │
       ▼ (Port 80)
+───────────────────────+
| Frontend UI (Nginx)   |
+───────────────────────+
       │
       ▼ (Internal DNS: http://backend:5000)
+───────────────────────+
| Backend API (Node/Py) |
+───────────────────────+
       │
       ▼ (Internal DNS: db:3306)
+───────────────────────+
| Database (MySQL)      | <───► [ Persistent Volume: mysql_data ]
+───────────────────────+
```

### Complete `docker-compose.yml`:
```yaml
services:
  db:
    image: mysql:8.0
    container_name: production_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: secretpassword
      MYSQL_DATABASE: shopdb
    volumes:
      - db_storage:/var/lib/mysql
    networks:
      - internal_network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-psecretpassword"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build: ./backend
    container_name: api_service
    restart: always
    environment:
      DB_HOST: db
      DB_NAME: shopdb
      DB_USER: root
      DB_PASS: secretpassword
    networks:
      - internal_network
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build: ./frontend
    container_name: ui_service
    restart: always
    ports:
      - "80:80"
    networks:
      - internal_network
    depends_on:
      - backend

networks:
  internal_network:
    driver: bridge

volumes:
  db_storage:
```

---

## 🧹 Step 13: Cleanup, Pruning & Maintenance

### ❓ Question 21: How do you safely clean up stopped containers, dangling images, and reclaim space?

```bash
# 1. Stop all running containers
docker stop $(docker ps -q)

# 2. Delete all stopped containers
docker rm $(docker ps -a -q)

# 3. Delete a specific container / image
docker rm <CONTAINER_ID_OR_NAME>
docker rmi <IMAGE_ID_OR_NAME>

# 4. Remove all dangling images (untagged <none>)
docker image prune

# 5. Nuclear cleanup: Reclaim ALL unused images, stopped containers, unused networks & build cache
docker system prune -a --volumes
```

---

## 📋 Quick Reference Cheat Sheet

| Task | Command |
| :--- | :--- |
| **Check Docker version** | `docker version` |
| **Build image with tag** | `docker build -t my-app:1.0 .` |
| **Run in background with port mapping** | `docker run -d -p 8080:80 --name my-app my-app:1.0` |
| **Inspect running container logs** | `docker logs -f my-app` |
| **Open interactive shell inside container**| `docker exec -it my-app /bin/sh` |
| **Inspect exact environment variables** | `docker inspect --format='{{json .Config.Env}}' my-app` |
| **View disk usage breakdown** | `docker system df -v` |
| **Start / Stop Docker Compose app** | `docker compose up -d` / `docker compose down` |
