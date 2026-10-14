# Docker Image Optimization

This repository is a practical guide for learning how to optimize Docker images: reduce size, improve build speed, improve security, and make images easier to deploy and maintain.

## Why optimize Docker images?

Optimizing a Docker image can help with:

- Smaller image size for faster downloads and lower storage costs
- Faster builds and deployments
- Better caching and improved developer productivity
- Reduced attack surface and lower security risk
- Cleaner, more maintainable Dockerfiles

## Core concepts

### 1. Use a minimal base image
Choose the smallest image that meets your runtime needs.

Examples:
- `alpine` for small Linux-based images
- `distroless` for production workloads with minimal dependencies
- `scratch` for fully static binaries

### 2. Use multi-stage builds
Build the app in one stage and copy only the final artifact into a minimal runtime image.

This is one of the most important optimization techniques for compiled applications.

### 3. Remove unnecessary files
Avoid copying source code, dependencies, tests, logs, build tools, or Git metadata into the final image.

Use a `.dockerignore` file to exclude non-essential files.

### 4. Keep layer order cache-friendly
Docker caches layers. Put stable, infrequently changed instructions earlier in the Dockerfile, and frequently modified instructions later.

Good order:
- install system dependencies
- copy dependency manifests
- install app dependencies
- copy source code
- build the app

### 5. Combine commands
Use a single `RUN` instruction to combine package installation and cleanup, reducing layers and keeping the final image smaller.

### 6. Avoid installing build tools in the final image
A builder stage can install compilers and dependencies, then copy only the resulting binary into a lean runtime image.

### 7. Scan for vulnerabilities and fix low-quality layers
Regular security scans help identify outdated packages and CVEs in your image.

## Example optimized Dockerfile

```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o app .

# Runtime stage
FROM alpine:3.20
WORKDIR /app
COPY --from=builder /app/app /app/app
EXPOSE 8080
CMD ["./app"]
```

This example reduces the image footprint by:
- compiling in a build stage
- using a lighter runtime base image
- copying only the final executable

## Useful optimization techniques

- Use `.dockerignore` to exclude unnecessary files
- Avoid `COPY . .` too early if only a few files are needed
- Use `--no-install-recommends` for Debian/Ubuntu package installs
- Remove apt cache after installation
- Use static binaries where possible
- Minimize package installation to only what the app actually needs
- Prefer small runtime images such as Alpine, Distroless, or Scratch

## Example commands

```bash
# Build image
docker build -t myapp:latest .

# Inspect layers
docker history myapp:latest

# Check image size
docker images myapp:latest

# Run the container
docker run --rm -p 8080:8080 myapp:latest
```

## Tools for image optimization

- `docker history` — inspect image layers
- `docker images` — compare image sizes
- `dive` — visualize image layers and biggest space consumers
- `trivy` — scan for vulnerabilities
- `hadolint` — lint Dockerfiles
- `docker buildx` — modern build tooling with caching improvements

## Best practices checklist

- [ ] Use a minimal base image
- [ ] Use multi-stage builds
- [ ] Remove build dependencies from final image
- [ ] Keep layers cache-friendly
- [ ] Use `.dockerignore`
- [ ] Remove apt/yum cache and temporary files
- [ ] Scan image for vulnerabilities
- [ ] Measure size before and after optimization

## Suggested project structure

```text
.
├── Dockerfile
├── .dockerignore
├── src/
├── scripts/
├── docs/
└── README.md
```

## Conclusion

Docker image optimization is both an engineering and operational practice. The goal is not only to make the image smaller, but to build a faster, safer, and more maintainable deployment artifact.

This repository will serve as a practical reference for Dockerfile optimization techniques, examples, and comparison strategies.

---

This is a draft README intended to help explain how to optimize a Docker image in a repository focused on image size and performance improvements.