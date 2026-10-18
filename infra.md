# Infrastructure & Deployment Specification

**Document:** `design/infra.md`  
**Status:** Current Active Infrastructure  
**Scope:** Edge Hosting, Routing, Database, and CI/CD Pipeline  
**Last Updated:** October 2026  

---

## 1. Overview

The system uses an edge-first serverless architecture:
* **Frontend:** Single Page Application (SPA) hosted on Vercel.
* **Backend API:** Serverless V8 isolate compute on Cloudflare Workers with custom domain routing.
* **Database:** Serverless distributed SQLite on Cloudflare D1.
* **CI/CD:** Automated testing, database migrations, and deployments via GitHub Actions.

---

## 2. Environments & Routing

| Layer | Platform | Environment | Domain / Identifier | Routing Details |
| :--- | :--- | :--- | :--- | :--- |
| **Frontend** | Vercel | Production / Active | `brain.mohakapoor.in` | SPA fallback rewrite (`/(.*) -> /index.html`) |
| **Backend API** | Cloudflare Workers | Dev / Active | `dev.mohakapoor.in` | API prefix: `/api_brain/*` |
| **Database** | Cloudflare D1 | Dev / Active | `brain-dev` | SQLite at Cloudflare edge |

---

## 3. CI/CD Deployment Pipeline

The backend repository utilizes GitHub Actions to automate deployment on pushes to the `dev` branch.

### Pipeline Stages
1. **Change Detection:**  
   Path filtering ensures CI only runs when backend code, configuration, or migrations change.
2. **Type Checking:**  
   Runs `tsc --noEmit` to validate strict TypeScript types before deployment.
3. **Database Migrations:**  
   Executes `wrangler d1 migrations apply brain-dev --remote` to apply pending schema changes automatically.
4. **Worker Deployment:**  
   Deploys the updated worker code to Cloudflare's edge network.
