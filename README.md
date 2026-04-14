# build & break

A running log of things I figured out while building personal projects and shipping hackathon prototypes. Less tutorial, more "here's exactly what tripped me up and why."

---

## What's in here

### [Deployment](./deployment/)

Getting code off your laptop and onto the internet — CI/CD, cloud platforms, environment config.

| Guide | Summary |
|-------|---------|
| [Azure App Service — Node.js + React](./deployment/azure-app-service.md) | Full-stack JS on Azure App Service via GitHub Actions. Covers Express serving the frontend, startup config, `/home` persistence trick, and student credits. |
| [Azure Web App for Containers — Docker](./deployment/azure-docker-web-app.md) | Another app but Dockerised — multi-stage build, ghcr.io registry, service principal auth. Covers `WEBSITES_PORT`, why Container Apps doesn't work on student subscriptions, and custom domain setup. |

---

## Philosophy

These notes are written for me-six-months-from-now who has forgotten the details. They skip the basics and go straight to the parts that weren't obvious — the config flags, the gotchas, the "why does this work" explanations.

If something here helps you too, great.
