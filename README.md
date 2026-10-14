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

Recommended next steps

- Add your site source under `src/` and set the correct build command in the Dockerfile
- Pin the `node` and `nginx` image tags you use for reproducible builds (e.g., `node:20-alpine`, `nginx:1.25.2-alpine`)
- Keep `.dockerignore` trimmed to reduce build context

