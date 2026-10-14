# Docker Image Optimization — Multi-stage build for a Simple Static Website (nginx)

This repository demonstrates a minimal, optimized multi-stage Dockerfile to build and serve a simple static website. The multi-stage approach keeps build tools out of the final image and produces a small runtime image containing only nginx and the built static files.

Project layout (recommended)

.
├── site/                # source files (if no build step)
├── src/                 # source files for sites that require a build step (e.g., Node)
├── Dockerfile
├── .dockerignore
└── README.md

Multi-stage Dockerfile (example with Node build stage)

```dockerfile
# Build stage: install dependencies and build static assets
FROM node:20-alpine AS build
WORKDIR /src
COPY package.json package-lock.json ./
RUN npm ci --silent
COPY . ./
# Replace 'build' with your project's build command; output should go to /src/dist
RUN npm run build

# Runtime stage: lightweight nginx image containing only the built assets
FROM nginx:1.25.2-alpine
# Remove default nginx content (optional)
RUN rm -rf /usr/share/nginx/html/*
# Copy built static files from the build stage
COPY --from=build /src/dist/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

If your site does not require a build step, you can skip the build stage and copy `site/` directly into the nginx image.

Example .dockerignore

```
node_modules/
.DS_Store
.git
.gitignore
dist/
README.md
Dockerfile
*.log
```

Build & run

```bash
# From the repository root, build the image
docker build -t static-site:latest .

# Run locally and map port 8080 to container port 80
docker run --rm -p 8080:80 static-site:latest
# Visit: http://localhost:8080
```

Why multi-stage?

- Keeps Node and build tooling out of the final image
- Final image contains only nginx + static assets — smaller and more secure
- Faster deployment and lower storage/transfer costs


## Compare image sizes: single-stage vs multi-stage

The best way to validate optimization is to build a "normal" (single-stage) image and a multi-stage (optimized) image, then compare their sizes and layer contents.

1. Create a simple single-stage Dockerfile (Dockerfile.single) that copies source/build output directly into nginx. Example:

```dockerfile
# Dockerfile.single
FROM node:20-alpine AS build
WORKDIR /src
COPY package.json package-lock.json ./
RUN npm ci --silent
COPY . ./
RUN npm run build

FROM nginx:1.25.2-alpine
COPY /src/dist/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Note: This single-stage example still uses two FROM lines but does not separate the build and runtime in the same way — depending on your Dockerfile this may include extra artifacts.

2. Build both images:

```bash
# Build single-stage (using Dockerfile.single)
docker build -f Dockerfile.single -t static-single:latest .

# Build multi-stage (using Dockerfile)
docker build -t static-multi:latest .
```

3. Quick size comparison (human-readable):

```bash
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" static-single:latest static-multi:latest
```

4. Precise size in bytes (scriptable):

```bash
# Print image size in bytes and convert to human-readable
echo "Single-stage: $(docker image inspect --format='{{.Size}}' static-single:latest) bytes"
echo "Multi-stage: $(docker image inspect --format='{{.Size}}' static-multi:latest) bytes"

# Optionally convert to human-readable with numfmt (Linux/macOS with coreutils)
numfmt --to=iec-i --suffix=B $(docker image inspect --format='{{.Size}}' static-single:latest)
numfmt --to=iec-i --suffix=B $(docker image inspect --format='{{.Size}}' static-multi:latest)
```

5. Inspect layer-by-layer differences:

- Use `docker history` to see per-layer sizes

```bash
docker history --no-trunc static-single:latest
docker history --no-trunc static-multi:latest
```

- Use `dive` to visualize layer contents and identify large files (recommended):

```bash
dive static-single:latest
dive static-multi:latest
```

6. Interpret results

- The multi-stage image should be smaller because it doesn't include build tools, dependency caches, or intermediate artifacts.
- `docker history` and `dive` will show which layers and files contribute most to size — focus optimization on those (e.g., removing node_modules from final image, cleaning caches, copying only dist).

7. Clean up

```bash
docker rmi static-single:latest static-multi:latest
```

Tips

- For repeatable comparisons pin image tags (e.g., `node:20-alpine`, `nginx:1.25.2-alpine`).
- If you use BuildKit, the build behavior can differ slightly; use the same build flags for both images when comparing (e.g., `DOCKER_BUILDKIT=1 docker build ...`).
- When measuring in CI, prefer `docker image inspect` for stable byte counts.

Recommended next step: run the two builds with your actual site sources and compare the outputs. The commands above let you quantify the size savings and identify which layers to optimize further.
