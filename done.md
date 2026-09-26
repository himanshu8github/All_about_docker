# ✅ Docker Learning Progress Tracker (`done.md`)

This file tracks every topic, experiment, and under-the-hood concept mastered so far.

---

## 🟢 Completed Steps & Concepts

### 1. Step 1: Docker Architecture & Core Mental Model
* [x] **Client-Server Architecture:** Understood the difference between Docker Client (`docker` CLI) and Docker Daemon (`dockerd` engine).
* [x] **Linux Primitives (Namespaces):** Proved how `pid`, `uts`, `mnt`, and `net` isolate processes.
* [x] **Kernel Sharing:** Verified that containers do not run their own OS kernel (ran Alpine Linux userland directly on an Ubuntu EC2 kernel).
* [x] **The PID 1 Rule:** Understood why containers die when PID 1 (`sh`) terminates, and why subsequent commands (`ps aux`) get auto-incrementing PIDs (12, 13, 14, 15...).
* [x] **Container States:** Demonstrated that stopped containers (`docker ps -a`) are NOT deleted and can be restarted (`docker start -ai`).

### 2. Step 2: Images vs Containers (Under the Hood)
* [x] **Read-Only Layers:** Learned that Docker images are composed of immutable, cryptographic SHA-256 layers.
* [x] **Container Writable Layer & UnionFS:** Understood how Docker slaps a thin read-write layer on top of read-only image layers.
* [x] **Copy-on-Write (CoW) in Action:** Used `docker diff` to catch file changes (`A` for Added, `C` for Changed) without modifying the base image.

### 3. Step 3: Disk Usage, Sizing & Under-the-Hood Inspection
* [x] **Container Size vs Virtual Size:** Ran `docker ps -a -s` to observe writable layer bytes vs total virtual image size.
* [x] **Storage Footprint:** Used `docker system df` to inspect disk usage across Images, Containers, Volumes, and Build Cache.
* [x] **Layer Inspection:** Used `docker history alpine` to examine layer creation commands and sizes.
* [x] **Byte Inspection:** Extracted raw image size in bytes using `docker image inspect --format='{{.Size}} bytes'`.

### 4. Real-World Debugging & Maintenance Skills Mastered
* [x] **Image Tags & Shared IDs:** Discovered why multiple tags (`himanshu/nginx:latest`, `nginx:latest`, `nginx:mainline`) share the exact same Image ID (`d0d674272be3`) without taking extra disk space.
* [x] **Untagging vs Layer Deletion:** Observed how Docker reference counting untags nicknames first and only deletes layers when the last tag is removed.
* [x] **Under-the-Hood Volume Inspection:** Inspected raw Docker volume mountpoints on the host (`/var/lib/docker/volumes/.../_data`).
* [x] **Production Database Recovery:** Mounted an orphaned MySQL volume to a temporary container, navigated the storage engine, and discovered database name (`message_db`) and table (`messages.ibd`).
* [x] **Docker CLI Grammar:** Mastered the `docker <noun/object> <verb/action>` syntax (e.g., `docker image prune`, `docker volume rm`).

---

## ⏳ Next Up

* [ ] **Step 4:** Container Lifecycle, Detached Mode (`-d`), and Port Binding Mechanics (`-p host:container`) with Nginx.
* [ ] **Step 5:** Inspecting Environment Variables, container metadata, and passing runtime overrides (`-e`, `--env-file`).
* [ ] **Step 6:** Dockerfile Deep Dive (`ENTRYPOINT` vs `CMD`, `COPY` vs `ADD`, `WORKDIR`, `HEALTHCHECK`).
* [ ] **Step 7:** Image Optimization & Multi-Stage Builds (dropping 1GB images to 20MB).
* [ ] **Step 8:** Persistent Storage: Named Volumes vs Host Bind Mounts.
* [ ] **Step 9:** Docker Networking: Bridge, Embedded DNS (`127.0.0.11`), and Round-Robin load balancing.
* [ ] **Step 10:** Docker Hub, Tags, and Registry Distribution.
* [ ] **Step 11:** Multi-Container Orchestration with Docker Compose v2.
* [ ] **Step 12:** Full-Stack Hands-On Project: MySQL + Backend + Frontend (Nginx).
* [ ] **Step 13:** Complete Production Maintenance & System Pruning.
