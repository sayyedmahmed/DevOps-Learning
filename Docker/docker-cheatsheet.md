# Docker Notes

A complete, organized cheat sheet covering Dockerfiles, image tagging, pushing, and container management — based on our session.

---

## Table of Contents

1. [Dockerfile — Core Instructions](#1-dockerfile--core-instructions)
2. [Dockerfile — Advanced Instructions](#2-dockerfile--advanced-instructions)
3. [Dockerfile — Best Practices](#3-dockerfile--best-practices)
4. [Full Dockerfile Examples](#4-full-dockerfile-examples)
5. [Image Naming & Tagging Rules](#5-image-naming--tagging-rules)
6. [Login & Push to Docker Hub](#6-login--push-to-docker-hub)
7. [Verify Remote Images](#7-verify-remote-images)
8. [Container Management](#8-container-management)
9. [Prune / Cleanup](#9-prune--cleanup)
10. [Inspect Image Contents](#10-inspect-image-contents)
11. [Sharing Your Image](#11-sharing-your-image)
12. [Common Errors & Fixes](#12-common-errors--fixes)
13. [Quick Reference Table](#13-quick-reference-table)

---

## 1. Dockerfile — Core Instructions

| Instruction | Purpose | Example |
| :--- | :--- | :--- |
| `FROM` | Base image (must be first) | `FROM ubuntu:22.04` |
| `RUN` | Execute at build time | `RUN apt-get update && apt-get install -y curl` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy local files into image | `COPY hello.py .` |
| `CMD` | Default command at runtime (overridable) | `CMD ["python3", "hello.py"]` |

### Basic example

```dockerfile
FROM ubuntu:22.04
WORKDIR /app
RUN apt-get update && apt-get install -y python3
COPY hello.py .
CMD ["python3", "hello.py"]
```

---

## 2. Dockerfile — Advanced Instructions

### `ENTRYPOINT` + `CMD`

`ENTRYPOINT` = fixed executable. `CMD` = default args (overridable at `docker run`).

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
ENTRYPOINT ["python3", "app.py"]
CMD ["--port", "8080"]
```

```bash
docker run myapp --port 9090   # overrides CMD only
```

### `ENV` vs `ARG`

- `ARG` → build-time only, not in final image
- `ENV` → persists at container runtime

```dockerfile
FROM alpine:3.19
ARG APP_VERSION=1.0          # build-time
ENV APP_HOME=/opt/app        # runtime
RUN echo "Building version $APP_VERSION"
WORKDIR $APP_HOME
```

```bash
docker build --build-arg APP_VERSION=2.0 -t myapp .
```

### `EXPOSE`

- **Documentation only** — does NOT publish the port.
- Still requires `-p` at runtime.

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

```bash
docker run -p 8080:80 mynginx
```

Multiple ports: `EXPOSE 80 443`

### `USER`

- Switches user for all subsequent instructions and runtime.
- **No password needed** — Docker sets UID/GID directly, no login/PAM.
- The user must be created first.

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .

RUN addgroup -S app && adduser -S app -G app
USER app

EXPOSE 3000
CMD ["node", "server.js"]
```

### `USER` — purpose & gotchas

| | Root (default) | `USER appuser` |
| :--- | :--- | :--- |
| Password | none | none |
| Privileges | full | limited |
| Bind port < 1024 | yes | no |
| Write system files | yes | no |
| Risk if compromised | high | low |

**Gotchas:**
- Files copied by `COPY` are owned by root → use `COPY --chown=appuser:appuser . .`
- `WORKDIR` created before `USER` is root-owned → `chown` it
- No home dir unless `useradd -m`
- No sudo; switch back with `USER root` (build time only)

**Fixed ownership example:**

```dockerfile
FROM python:3.11-slim
WORKDIR /app

RUN useradd -m appuser

COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . .

USER appuser
EXPOSE 5000
CMD ["python", "app.py"]
```

---

## 3. Dockerfile — Best Practices

### `.dockerignore`

```text
node_modules
.git
.env
*.log
Dockerfile
README.md
```

### Merge `RUN` commands (fewer layers)

```dockerfile
# ❌ 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ 1 layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

### Cache-friendly ordering

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./       # deps change rarely → cached
RUN npm ci
COPY . .                    # source changes often → busts only here
CMD ["node", "index.js"]
```

### Multi-stage build (smaller final image)

```dockerfile
# Stage 1: build
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o /out/app .

# Stage 2: runtime (tiny)
FROM alpine:3.19
COPY --from=builder /out/app /usr/local/bin/app
USER nobody
ENTRYPOINT ["app"]
```

---

## 4. Full Dockerfile Examples

### Node.js (production, non-root)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --chown=node:node . .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

### Python

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN useradd -m appuser
COPY --chown=appuser:appuser requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appuser . .
USER appuser
EXPOSE 5000
CMD ["python", "app.py"]
```

### PHP + Apache

```dockerfile
FROM php:8.2-apache
WORKDIR /var/www/html
COPY . .
RUN docker-php-ext-install pdo pdo_mysql
EXPOSE 80
```

---

## 5. Image Naming & Tagging Rules

### Format

```text
username / repository : tag
        └─ one part ─┘  └one word┘
```

- **Only ONE colon** total.
- Tag cannot contain `:` or `/`.
- Repository names must be **lowercase**; allowed: `a-z`, `0-9`, `-`, `_`, `.`

### Examples

| Command | Result |
| :--- | :--- |
| `docker tag php-app-image:v1 sayyedmahmed1/docker-projects:v1` | tag = `v1` |
| `docker tag php-app-image:v1 sayyedmahmed1/docker-projects:php-app-image` | tag = `php-app-image` |
| `docker tag php-app-image:v1 sayyedmahmed1/docker-projects:php-app-image-v1` | tag = `php-app-image-v1` |
| `...docker-projects:php-app-image:v1` | ❌ **invalid** (two colons) |

### Push multiple tags (same image, multiple labels)

```bash
docker tag php-app-image:v1 sayyedmahmed1/docker-projects:php-app-image
docker tag php-app-image:v1 sayyedmahmed1/docker-projects:v1
docker push sayyedmahmed1/docker-projects:php-app-image
docker push sayyedmahmed1/docker-projects:v1
```

### Repository vs Tag

- **Repository** = the "folder" (e.g., `docker-projects`) — cannot be renamed after creation.
- **Tag** = the "file/version" inside (e.g., `v1`, `php-app-image-v1`).
- To use a different repository name → create a new repo and re-push.

---

## 6. Login & Push to Docker Hub

### Login (terminal only — skip browser flow)

```bash
docker login -u sayyedmahmed1
# then type password / PAT at prompt
```

Safer (no password in history):

```bash
docker login -u sayyedmahmed1 --password-stdin
```

### Push

```bash
docker tag php-app-image:v1 sayyedmahmed1/docker-projects:php-app-image-v1
docker push sayyedmahmed1/docker-projects:php-app-image-v1
```

Expected output ends with:

```text
php-app-image-v1: digest: sha256:... size: ...
```

---

## 7. Verify Remote Images

### List all tags via API

```bash
curl -s "https://hub.docker.com/v2/namespaces/sayyedmahmed1/repositories/docker-projects/tags?page_size=100" \
  | jq -r '.results[].name'
```

### Or with `skopeo`

```bash
skopeo list-tags docker://docker.io/sayyedmahmed1/docker-projects
```

### Or in browser

```text
https://hub.docker.com/r/sayyedmahmed1/docker-projects/tags
```

---

## 8. Container Management

### List

| Command | Shows |
| :--- | :--- |
| `docker ps` | running only |
| `docker ps -a` | all (running + stopped) |
| `docker ps -q` | running IDs only |
| `docker ps -a -q` | all IDs only |

Clean custom table:

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
```

### Stop & delete

| Task | Command |
| :--- | :--- |
| Stop one | `docker stop <name>` |
| Delete stopped | `docker rm <name>` |
| Stop + delete | `docker rm -f <name>` |
| Stop all running | `docker stop $(docker ps -q)` |
| Delete all stopped | `docker rm $(docker ps -a -q)` |
| Nuke all containers | `docker rm -f $(docker ps -a -q)` |

### Logs / stats / inspect

```bash
docker logs <container>
docker stats
docker inspect <container>
```

---

## 9. Prune / Cleanup

| Command | Removes |
| :--- | :--- |
| `docker container prune` | all **stopped** containers (running safe) |
| `docker image prune` | dangling images only |
| `docker image prune -a` | all images not used by a container |
| `docker system prune` | stopped containers + unused networks + dangling images + build cache |
| `docker system prune --volumes` | also unused anonymous volumes |

- All prompt for confirmation; add `-f` to skip.
- `docker system prune` **does not** remove named volumes by default.

---

## 10. Inspect Image Contents

```bash
# Pull first
docker pull sayyedmahmed1/docker-projects:php-app-image-v1

# List files without starting a container
docker run --rm sayyedmahmed1/docker-projects:php-app-image-v1 ls -la /

# Interactive shell
docker run --rm -it sayyedmahmed1/docker-projects:php-app-image-v1 /bin/bash

# Export filesystem to tarball
docker create --name tmpinspect sayyedmahmed1/docker-projects:php-app-image-v1
docker export tmpinspect -o image-contents.tar
docker rm tmpinspect
tar -tf image-contents.tar | head -50

# Metadata
docker inspect sayyedmahmed1/docker-projects:php-app-image-v1

# Layer history
docker history sayyedmahmed1/docker-projects:php-app-image-v1
```

> Docker Hub web UI shows only tags/sizes/digests — **no file browser**.

---

## 11. Sharing Your Image

### Reference

```text
sayyedmahmed1/docker-projects:php-app-image-v1
```

### Link

```text
https://hub.docker.com/r/sayyedmahmed1/docker-projects
```

### Others pull & run

```bash
docker pull sayyedmahmed1/docker-projects:php-app-image-v1
docker run sayyedmahmed1/docker-projects:php-app-image-v1
```

> Ensure repo visibility is **Public** in Docker Hub → Settings.

---

## 12. Common Errors & Fixes

| Error | Cause | Fix |
| :--- | :--- | :--- |
| `invalid reference format` | Two colons in image name | Use only one `:` → `repo:tag` |
| `image is being used by a container` | Container references it | Stop/remove container, or `docker rmi -f` |
| `jq: syntax error` | Extra bracket in filter | `jq -r '.results[].name'` (no trailing `]`) |
| Browser login stuck | Docker defaults to web auth | `docker login -u <username>` |
| Push says repo not found | Not tagged with target name | `docker tag ... username/repo:tag` first |
| Permission denied in container | Files owned by root | `COPY --chown=user:user` |

---

## 13. Quick Reference Table

| Task | Command |
| :--- | :--- |
| Build image | `docker build -t name:tag .` |
| List local images | `docker images` |
| Tag image | `docker tag src:tag user/repo:tag` |
| Login (terminal) | `docker login -u user` |
| Push | `docker push user/repo:tag` |
| Pull | `docker pull user/repo:tag` |
| Run detached | `docker run -d --name c -p 8080:80 user/repo:tag` |
| List running | `docker ps` |
| Stop | `docker stop <name>` |
| Remove container | `docker rm -f <name>` |
| Remove image | `docker rmi user/repo:tag` |
| Clean stopped containers | `docker container prune` |
| Clean everything unused | `docker system prune` |
| Inspect image | `docker inspect user/repo:tag` |
| Export image contents | `docker export <container> -o out.tar` |

