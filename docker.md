# Docker for DevOps Interviews (4+ Years Experience)

## 1. Basic Syntax

**Basic Dockerfile and Instructions**

```dockerfile
# Base image
FROM ubuntu:22.04

# Set working directory
WORKDIR /app

# Set environment variables
ENV APP_PORT=8080

# Install dependencies
RUN apt-get update && apt-get install -y curl python3

# Copy application code
COPY src/ /app/src/

# Expose port (Documentation purpose)
EXPOSE $APP_PORT

# Default command
CMD ["python3", "-m", "http.server", "8080"]
```

---

## 2. Intermediate Examples

**Build Arguments, Entrypoints, and Docker Compose**

*Dockerfile with ARG and ENTRYPOINT*
```dockerfile
FROM alpine:3.18

# ARG is available only during build time
ARG VERSION=1.0.0
RUN echo "Building app version $VERSION" > /version.txt

# RUN layer caching optimization (combine apt/apk commands)
RUN apk add --no-cache bash curl

COPY start.sh /start.sh
RUN chmod +x /start.sh

# ENTRYPOINT makes the container act like an executable
ENTRYPOINT ["/start.sh"]
# CMD is passed as an argument to ENTRYPOINT
CMD ["--default-mode"]
```

*Docker Compose (`docker-compose.yml`)*
```yaml
version: '3.8'
services:
  web:
    build: 
      context: .
      args:
        VERSION: 2.0.0
    ports:
      - "80:8080"
    environment:
      - DB_HOST=db
    depends_on:
      - db
    networks:
      - backend-network

  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret
    networks:
      - backend-network

volumes:
  db-data:

networks:
  backend-network:
```

---

## 3. Advanced Examples

**Multi-Stage Builds, Security, and Non-Root Users**

*Production-ready Multi-stage Golang Build*
```dockerfile
# --- Stage 1: Build ---
FROM golang:1.21-alpine AS builder

WORKDIR /app
# Copy mod files first for layer caching
COPY go.mod go.sum ./
RUN go mod download

COPY . .
# Build statically linked binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# --- Stage 2: Run ---
FROM scratch AS production

# Copy compiled binary from builder stage
COPY --from=builder /app/main /main
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Run as non-root user (security best practice)
# (Assuming a user with uid 1000 was created or exists in a base image)
USER 1000:1000

# Implement healthcheck
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD ["/main", "-healthcheck"]

EXPOSE 8080
ENTRYPOINT ["/main"]
```
*Why this is advanced:* Multi-stage builds reduce the final image size drastically (e.g., 800MB -> 15MB) and remove build tools (reducing attack surface). `scratch` is an empty image.

---

## 4. Interview Coding Exercises

### Problem 1: Dockerfile Layer Caching
**Task:** Review this Node.js Dockerfile. How would you optimize it for faster rebuilds when only the source code changes?
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
```
**Solution:**
```dockerfile
FROM node:18
WORKDIR /app
# Copy package files first
COPY package*.json ./
# Run install. This layer is cached unless package.json changes
RUN npm install
# Now copy source code
COPY . .
CMD ["npm", "start"]
```

### Problem 2: Volume Mounting
**Task:** Write a Docker command to run an `nginx` container, mapping port 80 to host port 8080, and mounting the local `./html` directory to `/usr/share/nginx/html`.
**Solution:**
```bash
docker run -d -p 8080:80 -v $(pwd)/html:/usr/share/nginx/html nginx
```

---

## 5. Troubleshooting Exercises

### Broken Configuration 1: Container Exits Immediately
**Scenario:** You start a container that runs a bash script, but it immediately exits with status 0.
`docker run -d my-image`
**What is wrong?**
**Answer:** Containers exit when their primary PID 1 process finishes. If a script executes sequentially and ends, or runs a process in the background (e.g., `nginx &`), the script completes and the container dies. 
**Fix:** Keep the primary process in the foreground (e.g., use `nginx -g 'daemon off;'` or `tail -f /dev/null` at the end of the script).

### Broken Configuration 2: Orphaned Volumes
**Scenario:** A disk on a build server is full. You notice hundreds of GBs used by Docker volumes that are no longer attached to containers.
**Answer/Fix:** Run `docker system prune --volumes -f` (or specifically `docker volume prune -f`) to remove dangling volumes not attached to any container.

---

## 6. Common Interview Questions

**Q: "Explain the difference between COPY and ADD."**
*Answer:* Both copy files into the image. `COPY` is preferred for standard files/directories. `ADD` has extra features: it can automatically extract `.tar` files into the image, and it can fetch files from remote URLs. Best practice is to use `COPY` unless you explicitly need `ADD`'s extraction capability.

**Q: "What is the difference between CMD and ENTRYPOINT?"**
*Answer:* 
- `ENTRYPOINT` configures the container to run as an executable. It cannot be easily overridden at runtime (unless using `--entrypoint`).
- `CMD` sets default arguments or a default command. If `ENTRYPOINT` is defined, `CMD` acts as default arguments passed to `ENTRYPOINT`. `CMD` can easily be overridden by passing arguments at the end of `docker run`.

**Q: "How do you secure a Docker container in production?"**
*Answer:*
1. Do not run as root (`USER nonroot`).
2. Use minimal base images (Alpine, Distroless, Scratch).
3. Scan images for vulnerabilities (Trivy).
4. Drop Linux capabilities (`--cap-drop=ALL`).
5. Make the filesystem read-only (`--read-only`).
6. Do not hardcode secrets (use Secrets Manager/Vault).

---

## 7. Cheat Sheet

| Command | Usage (4+ Yrs Experience Focus) |
| :--- | :--- |
| `docker build -t app:v1 --no-cache .` | Force rebuild from scratch, ignoring layer cache. |
| `docker exec -it <container> /bin/sh` | Open a shell in a running container for debugging. |
| `docker inspect <container>` | Output detailed JSON configuration (IP address, mounts). |
| `docker logs --tail 100 -f <container>`| Follow the last 100 lines of container logs. |
| `docker system prune -a` | Clean up stopped containers, unused networks, and dangling images. |
| `docker image history <image>` | See the layers and size of each instruction in an image. |
