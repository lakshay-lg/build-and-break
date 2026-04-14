# Deploying a Dockerised Node.js + React App to Azure Web App for Containers

Same app as the [App Service guide](./azure-app-service.md) — Express backend serving a Vite/React frontend — but this time packaged as a Docker container and deployed via GitHub Container Registry. The architecture is identical; only the packaging and delivery change.

---

## Why Docker over Code Deploy?

The "Publish: Code" approach in App Service relies on Azure's build system (Kudu) understanding your project layout. The Docker approach gives you a self-contained artifact — the same image runs identically locally, in CI, and in production. If it works in the container on your machine, it works in Azure.

Also useful when you want to pin the exact Node version, add OS-level dependencies, or use a runtime Azure doesn't natively support.

---

## Architecture Overview

```
GitHub repo (main branch)
    ↓ push
GitHub Actions
    ↓ docker build (multi-stage)
    ↓ push image → ghcr.io/<username>/appname:sha
Azure Web App for Containers
    ↓ pulls image, restarts container
    ↓ Express serves API + React SPA on port 3001
Browser
```

One container, one service, one bill. Same as before — just with Docker in the middle.

---

## The Multi-Stage Dockerfile

The key insight: use two `FROM` instructions. Stage 1 builds the frontend (needs all devDependencies, Vite, Node). Stage 2 is the production image — it copies only the built `frontend/dist` from Stage 1. The 300MB+ of frontend `node_modules` never makes it into the final image.

```dockerfile
# Stage 1: Build React frontend
FROM node:22-alpine AS frontend-build
WORKDIR /app/frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

# Stage 2: Production image
FROM node:22-alpine AS production
WORKDIR /app

COPY backend/package*.json ./backend/
RUN cd backend && npm ci --omit=dev

COPY backend/ ./backend/
COPY --from=frontend-build /app/frontend/dist ./frontend/dist

ENV NODE_ENV=production
EXPOSE 3001

WORKDIR /app/backend
CMD ["node", "server.js"]
```

**`--omit=dev`** — installs only production dependencies in Stage 2. `npm ci --omit=dev` is the correct flag (not `--production`, which is deprecated).

**`COPY --from=frontend-build`** — this is how you cross stage boundaries. The `frontend-build` stage's filesystem is accessible even though it ran in a completely separate layer.

**Path assumption** — `server.js` uses `path.join(__dirname, "../frontend/dist")` for static files. The Dockerfile mirrors this: `backend/` and `frontend/dist` end up as siblings under `/app/`, so `__dirname` (`/app/backend`) + `../frontend/dist` resolves correctly.

---

## .dockerignore

Critical. Without it, Docker sends `node_modules` (hundreds of MB) as build context on every build. Add this at repo root:

```
node_modules
**/node_modules
frontend/dist
.git
.github
.env
.env.*
!.env.example
*.zip
docs
README.md
```

---

## GitHub Container Registry (ghcr.io)

ghcr.io is GitHub's built-in container registry. It's free and your GitHub Actions workflow already has credentials to push to it via the auto-injected `GITHUB_TOKEN` — no separate registry account needed.

The image name follows the pattern: `ghcr.io/<github-username>/<repo-or-image-name>:<tag>`

After the first push, the package appears under your GitHub profile → Packages. **It defaults to private.** Azure can't pull a private image without credentials, so either:
- Make the package public (GitHub profile → Packages → your image → Package settings → Change visibility)
- Or configure registry credentials in Azure App Settings

Making it public is simpler for side projects.

---

## GitHub Actions Workflow

Two jobs — build/push first, deploy after:

```yaml
name: Build and Deploy to Azure

on:
  push:
    branches: [main]

env:
  IMAGE_NAME: ${{ github.repository_owner }}/cardsense

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write        # needed to push to ghcr.io

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # auto-injected, no setup needed

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ env.IMAGE_NAME }}:latest
            ghcr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    needs: build-push

    steps:
      - uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
          images: ghcr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
```

**Tag with commit SHA, not just `latest`** — using `${{ github.sha }}` means each deploy pulls a specific, traceable image. If you only tag `latest`, Azure may skip a pull if it thinks it already has the latest.

**`permissions: packages: write`** — required explicitly on the `build-push` job for `GITHUB_TOKEN` to have push access to ghcr.io. Without this, the push silently fails.

---

## Azure Setup

### Don't use Azure Container Apps on student subscriptions

Container Apps (the serverless, scale-to-zero option) is not available on Azure for Students. The portal lets you fill out the whole form and fails at validation with `RequestDisallowedByAzure`. Changing regions doesn't fix it — it's a subscription-level policy restriction, not a regional availability issue.

**Use Web App for Containers instead** — same Docker workflow, runs on App Service, works on student subscriptions.

### Create the Web App

portal.azure.com → Web App → Create:

| Field | Value |
|-------|-------|
| Publish | **Docker Container** (not "Code") |
| Operating System | Linux |
| Pricing plan | B1 Basic (~$13/month) or F1 Free (no custom domain support) |

On the Docker tab: use a placeholder image (`mcr.microsoft.com/azuredocs/containerapps-helloworld:latest`) for now. GitHub Actions will replace it on first deploy.

### The critical env var: `WEBSITES_PORT`

Azure doesn't know which port your container listens on. You tell it via `WEBSITES_PORT`. If this is wrong or missing, Azure forwards traffic to the wrong port and you get 503s on a perfectly healthy container.

Set in Environment variables (not General settings):

| Name | Value |
|------|-------|
| `NODE_ENV` | `production` |
| `WEBSITES_PORT` | `3001` (match whatever `app.listen()` uses) |
| `ANTHROPIC_API_KEY` | your key — check "Deployment slot setting" for secrets |

### Authentication: service principal, not publish profile

Unlike the Code deploy approach (which uses a publish profile XML), Docker-based deploys authenticate via a service principal. Run this in Azure Cloud Shell (the `>_` button in the portal top bar — no local install needed):

```bash
az account show --query id -o tsv   # get your subscription ID

az ad sp create-for-rbac \
  --name myapp-gh-actions \
  --role contributor \
  --scopes /subscriptions/<SUB_ID>/resourceGroups/<YOUR_RG> \
  --sdk-auth
```

Copy the entire JSON output → GitHub repo → Settings → Secrets → `AZURE_CREDENTIALS`.

Also add `AZURE_WEBAPP_NAME` secret (the exact Web App name from the portal).

---

## Custom Domain

F1 (free) doesn't support custom domains — upgrade to B1 first.

For domains hosted elsewhere (e.g. name.com):

1. Azure portal → Web App → Custom domains → Add custom domain → select **"All other domain services"**
2. Note the **IP address** and **Custom Domain Verification ID** shown on the Custom Domains page
3. At your registrar, add:

**For a subdomain (`app.yourdomain.com`):**

| Type | Host | Value |
|------|------|-------|
| CNAME | `app` | `yourapp.azurewebsites.net` |
| TXT | `asuid.app` | the Verification ID |

**For root domain (`yourdomain.com`):**

| Type | Host | Value |
|------|------|-------|
| A | `@` | the IP address |
| TXT | `asuid` | the Verification ID |

4. Wait 5–10 min for DNS, then Validate in Azure → Add custom domain
5. Add binding → **App Service Managed Certificate** (free) → SNI SSL

---

## Updating the App

Just push to `main`. GitHub Actions builds a new image, pushes it, and Azure pulls and restarts:

```bash
git add .
git commit -m "your change"
git push origin main
```

Full pipeline takes ~3–5 minutes. Watch it under repo → Actions tab.

---

## Troubleshooting

### 503 on a healthy container
`WEBSITES_PORT` is wrong or not set. Verify it matches the port in `app.listen()`.

### Deploy job fails with auth error
- `AZURE_CREDENTIALS` secret is missing, malformed, or the service principal expired
- Re-run `az ad sp create-for-rbac` and update the secret

### Azure can't pull the image
- The ghcr.io package is private — make it public, or configure registry credentials in Azure App Settings
- Image name in the workflow doesn't match what was pushed (case-sensitive)

### Container starts but app crashes
- Check **Log stream** in Azure portal → Web App → Monitoring → Log stream
- `NODE_ENV` not set → server not serving frontend static files
- A dependency is missing from `backend/package.json` (was only in devDependencies)

### First deploy fails, subsequent ones work
Expected — the `deploy` job needs `AZURE_CREDENTIALS` to exist. If you push before adding secrets, the `build-push` job creates the image but `deploy` fails. Add secrets, then push an empty commit:
```bash
git commit --allow-empty -m "retrigger deploy"
git push origin main
```

---

## Cost vs Previous Approach

| | Code Deploy | Docker Deploy |
|--|-------------|---------------|
| Azure service | App Service (Code) | Web App for Containers |
| Pricing | Same B1 ~$13/month | Same B1 ~$13/month |
| Image storage | N/A | ghcr.io — free |
| Auth method | Publish profile XML | Service principal JSON |
| Build happens | In GitHub Actions | In GitHub Actions (Docker) |
| Reproducibility | Depends on Azure's build | Identical image everywhere |

The cost is identical. The Docker approach adds a build step but removes any "works on my machine" risk.
