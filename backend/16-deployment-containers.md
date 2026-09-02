# Backend Standards: Deployment & Container Conventions

Rules for building container images, keeping environments consistent, and deploying without downtime.

## 1. Dockerfile Conventions

*   **Multi-Stage Builds:** Use a `build` stage with full `devDependencies` to compile TypeScript, and a slim `runtime` stage that copies only the compiled output plus production dependencies. Never ship `devDependencies` or source `.ts` files in the final image.
*   **Pin the Base Image:** Reference a specific, pinned Node.js image tag (e.g. `node:20.11-slim`), not `latest` — an unpinned base image makes builds non-reproducible and can introduce breaking changes silently.
*   **Run as a Non-Root User:** The container must run the application as a dedicated non-root user, not `root`.
*   **Minimal Image Surface:** Use a slim/alpine base and avoid installing build tools or unnecessary OS packages in the final runtime stage.

```dockerfile
# Good: multi-stage build
FROM node:20.11-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20.11-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && addgroup --system app && adduser --system --ingroup app app
COPY --from=build /app/dist ./dist
USER app
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

---

## 2. Environment Parity

*   **Promote One Image Through Environments:** Build the container image once and promote the same artifact through dev → staging → production, rather than rebuilding per environment — configuration differences come from environment variables/secrets injected at deploy time, not from rebuilding with different code.
*   **Same Node Version Everywhere:** Local development, CI, and the container runtime should all target the same Node.js major version to avoid "works on my machine" discrepancies.

---

## 3. Zero-Downtime Deploys

*   **Readiness Gates Traffic:** The deployment orchestrator must only route traffic to a new instance once its readiness probe passes (DB connected, app fully bootstrapped) — see [10-observability-health.md](10-observability-health.md#2-readiness-vs-liveness).
*   **Graceful Shutdown on Rollout:** Outgoing instances must drain in-flight requests via the shutdown hooks described in [10-observability-health.md](10-observability-health.md#3-graceful-shutdown) before the orchestrator kills the process.
*   **Rolling, Not All-at-Once:** Deploy new instances gradually (rolling update) so a bad deploy affects only a fraction of traffic and can be caught before full rollout.

---

## 4. Secrets in Deployment

*   **Inject at Deploy Time, Never Bake into the Image:** Secrets and environment-specific config must be provided via the orchestrator's secret store/environment injection mechanism at container start — never `COPY`'d into the image or hardcoded in the Dockerfile. See [05-security-auth.md](05-security-auth.md#4-secrets--sensitive-files).
*   **No Secrets in Build Args or Layer History:** Avoid passing secrets as Docker `ARG`/`ENV` during the build stage — they persist in image layer history even if unset later.

---

## 5. Build Reproducibility

*   **Lockfile-Driven Installs:** CI and image builds must use `npm ci` (not `npm install`) so the exact versions in the committed lockfile are installed, guaranteeing reproducible builds.
*   **Commit the Lockfile:** `package-lock.json` (or equivalent) must be committed and kept in sync with `package.json` — never let it drift or get gitignored.
