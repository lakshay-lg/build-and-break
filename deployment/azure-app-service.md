# Deploying a Node.js + React App to Azure App Service via GitHub Actions

A general-purpose reference for deploying any full-stack JavaScript app (Express backend + Vite/React frontend) to Azure App Service using GitHub Actions for CI/CD.

---

## Architecture Overview

```
GitHub repo (main branch)
    ↓ push
GitHub Actions workflow
    ↓ builds frontend + installs backend deps
Azure App Service (Linux B1)
    ↓ Express serves both API and built React app
Browser
```

The key idea: instead of deploying frontend and backend as separate services, Express serves the built React app as static files in production. One service, one bill, one deployment.

---

## Prerequisites

- GitHub account with the repo you want to deploy
- Azure account (Azure for Students gives $100 free credit — no credit card needed)
- Node.js app with a clear backend entry point (e.g. `backend/server.js`)
- React/Vite frontend (or any static frontend with a build step)

---

## Step 1 — Modify Express to Serve the Frontend in Production

In your backend `server.js`, add a static file server that activates only in production:

```js
import { existsSync } from "fs";
import { resolve, dirname } from "path";
import { fileURLToPath } from "url";

const __dirname = dirname(fileURLToPath(import.meta.url));
const IS_PROD = process.env.NODE_ENV === "production";

// ... your existing routes ...

// Serve frontend in production (add this AFTER all API routes)
if (IS_PROD) {
  const frontendDist = resolve(__dirname, "../frontend/dist");
  if (existsSync(frontendDist)) {
    app.use(express.static(frontendDist));
    // SPA fallback — lets React Router handle client-side navigation
    app.get("*", (req, res) => {
      res.sendFile(resolve(frontendDist, "index.html"));
    });
  }
}
```

**Why the catch-all `*` route?** React Router handles navigation client-side. If a user visits `/policy/123` directly, the server needs to return `index.html` so React can take over — otherwise it 404s.

---

## Step 2 — Pin Your Node Version

Add an `engines` field to your backend `package.json`:

```json
{
  "engines": {
    "node": ">=22.0.0"
  }
}
```

This tells Azure (and anyone else) exactly what Node version your app expects, preventing silent failures from version mismatches.

---

## Step 3 — Create the GitHub Actions Workflow

Create `.github/workflows/azure-deploy.yml` in your repo root:

```yaml
name: Deploy to Azure

on:
  push:
    branches: [ main ]
  workflow_dispatch:        # allows manual trigger from GitHub UI

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install and build frontend
        run: |
          cd frontend
          npm ci
          npm run build

      - name: Install backend dependencies
        run: |
          cd backend
          npm ci

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
          package: .        # deploys entire repo root
```

**Why `npm ci` instead of `npm install`?** `ci` installs exactly what's in `package-lock.json` and errors if the lockfile is out of sync. This makes builds reproducible — you get the same versions every time, not whatever `npm install` resolves to that day.

---

## Step 4 — Create the Azure App Service

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search **App Services** → **Create** → **Web App**
3. Fill in:

| Field | Value |
|-------|-------|
| Subscription | Your subscription |
| Resource Group | Create new — e.g. `myapp-rg` |
| Name | `myapp` (becomes `myapp.azurewebsites.net`) |
| Publish | Code |
| Runtime stack | Node 22 LTS |
| Operating System | **Linux** (4x cheaper than Windows) |
| Region | Central India / South India (cheapest for INR) |
| Pricing plan | **Basic B1** |
| Zone redundancy | Disabled |

4. Skip the Database tab (unless you need Azure-managed DB)
5. Click through Networking and Monitoring without changes
6. **Review + Create** → **Create**

### Pricing reference

| Plan | Cost (Linux) | RAM | CPU | Use case |
|------|-------------|-----|-----|----------|
| F1 Free | $0 | 1 GB shared | 60 min/day | Not viable for AI/compute-heavy apps |
| B1 Basic | ~$13-14/month | 1.75 GB | 1 vCPU dedicated | Good for most side projects |
| B2 Basic | ~$27/month | 3.5 GB | 2 vCPU | If you need more memory |

**Avoid Windows plans** — they cost ~4x more for equivalent specs.

---

## Step 5 — Configure the App Service

### Enable Basic Authentication (needed for publish profile)

Azure disables this by default. Go to:

**Settings → Configuration → General settings** → enable:
- SCM Basic Auth Publishing Credentials → **On**
- FTP Basic Auth Publishing Credentials → **On**

Save.

### Set the Startup Command

In the same **General settings** tab, set:

```
node backend/server.js
```

Adjust the path to match your actual entry point. Without this, Azure may run the wrong file.

### Set Environment Variables

**Settings → Environment variables** → add:

| Variable | Value |
|----------|-------|
| `NODE_ENV` | `production` |
| `PORT` | `8080` |
| `YOUR_API_KEY` | your actual key |
| `DB_FILE` | `/home/yourapp-db.json` ← see note below |

**The `/home` persistence trick:** Azure App Service Linux persists the `/home` directory across restarts and redeployments (it's backed by Azure Files network storage). Everything else is ephemeral — wiped on restart. If you're using a flat-file database (JSON, SQLite), point it to `/home/` to survive redeployments. This is unique to Azure App Service and not available on most other PaaS platforms.

---

## Step 6 — Connect GitHub and Get the Publish Profile

### Download the publish profile

In your App Service → **Overview** → click **Download publish profile**. This downloads a `.PublishSettings` XML file containing your deployment credentials.

### Add GitHub secrets

In your GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

| Secret name | Value |
|-------------|-------|
| `AZURE_WEBAPP_NAME` | your app name (e.g. `myapp`) |
| `AZURE_PUBLISH_PROFILE` | entire contents of the `.PublishSettings` XML file |

**Select all** the XML content (`Ctrl+A`) and paste it in full. Partial pastes will cause "No credentials found" errors.

---

## Step 7 — Deploy

Push to `main`:

```bash
git add .
git commit -m "add azure deployment config"
git push origin main
```

GitHub Actions triggers automatically. Watch it run under your repo's **Actions** tab. First deploy takes 3-5 minutes.

To manually trigger without a code change:

```bash
git commit --allow-empty -m "retrigger deployment"
git push origin main
```

---

## Step 8 — Verify

```
https://yourapp.azurewebsites.net/api/health   ← should return your health check JSON
https://yourapp.azurewebsites.net              ← should serve your React frontend
```

---

## Troubleshooting

### "No credentials found" in GitHub Actions
- The `AZURE_PUBLISH_PROFILE` secret is missing, empty, or wasn't pasted in full
- Check that Basic Auth is enabled in Azure Configuration
- Delete and re-add the secret

### Frontend loads but API calls fail
- Check that API routes are defined **before** the static file middleware in `server.js`
- Verify `NODE_ENV=production` is set in Azure environment variables
- Check the startup command is set correctly

### Policies/data not persisting after redeploy
- Make sure `DB_FILE` points to `/home/yourfile.json`, not a relative path
- Relative paths resolve inside the app directory which is ephemeral

### "Dangerous site" browser warning
- This is a Google Safe Browsing false positive common with new `azurewebsites.net` subdomains
- Submit for review at `safebrowsing.google.com/safebrowsing/report_error/`
- Long-term fix: add a custom domain

### App works locally but crashes on Azure
- Check Node version — Azure may default to an older version than you expect
- Verify all environment variables are set (Azure doesn't read your local `.env`)
- Check logs: **App Service → Monitoring → Log stream**

---

## .gitignore Essentials

```gitignore
# Never commit these
node_modules/
.env
backend/.env
frontend/.env

# Azure/tool-specific (protect even if global git config covers it)
.claude/

# Frontend build — GitHub Actions rebuilds this fresh every deploy
frontend/dist/
dist/
```

---

## GitHub Student Pack — What Actually Works for Deployment

Verified as of April 2026:

| Platform | Student Pack Benefit | Notes |
|----------|---------------------|-------|
| **Azure** | $100 credit (Azure for Students) | Sign up with university email, no credit card |
| **DigitalOcean** | $200 credit for 1 year | Also via Student Pack |
| **Railway** | None confirmed | Not in Student Pack despite some claims |
| **Vercel** | Free tier (no Student Pack needed) | Best for frontend-only |
| **Koyeb** | Free always-on tier | Good free backend option, ephemeral disk |
| **Render** | Free tier spins down after 15 min | Usable for low-traffic/demos |

---

## Cost Estimate (Azure B1 Linux, Central India)

| Resource | Monthly cost |
|----------|-------------|
| App Service B1 Linux | ~$13-14 |
| Outbound bandwidth (first 5 GB) | $0 |
| **Total** | **~$13-14/month** |

With $100 Azure for Students credit: approximately **7 months** of free hosting.
