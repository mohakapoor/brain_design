# High-Level Design (HLD) — Personal Second Brain & Execution System

**Document:** `design/hld.md`  
**Status:** Approved Architectural Baseline  
**Scope:** End-to-End System Architecture (Frontend, Edge Backend, D1 Persistence, Dual-Auth, MCP AI Layer)  
**Last Updated:** October 2026  

---

## 1. Executive Summary & Core Philosophy

The **Personal Second Brain & Execution System** is a low-latency, edge-first task capture and execution platform designed around a single psychological principle:

> **Working memory is for processing, not storage.** While working, new thoughts, tasks, and ideas emerge constantly. Capturing them must have lower cognitive friction than attempting to remember them.

The system's core operating loop is:
```text
Capture ──► Review ──► Execute ──► Track ──► Improve ──↺ (Loops back to Capture)
```

### 1.1 Architectural Axioms
1. **Execution Over Endless Planning:** Zero premature planning bureaucracy. The system facilitates immediate capture, single-task execution focus, and velocity tracking.
2. **Two Interfaces, One Source of Truth:**
   * **Web Dashboard:** A high-craft visual interface (React 19 + Vite) for manual capture, task triage, metrics, and velocity analysis.
   * **Conversational AI Agent (MCP):** A headless Model Context Protocol (MCP) server allowing clients (Claude Desktop, Cursor, ChatGPT) to manipulate tasks via natural language.
   * **Rule:** Both interfaces query the identical Cloudflare Worker backend and Cloudflare D1 SQLite database. The AI layer is never a separate data silo.
3. **Sub-Millisecond Edge Architecture:** All computing runs on V8 isolates via Cloudflare Workers; relational data resides in Cloudflare D1 distributed SQLite.
4. **Dual-Model Authentication:**
   * Web browsers authenticate via secure, `HttpOnly` database-backed session cookies.
   * Headless AI / MCP clients authenticate via `Authorization: Bearer <api_key>` backed by SHA-256 hashed API keys.

---

## 2. System Architecture Diagram

```mermaid
graph LR
    Frontend["Frontend (Web App)"] -->|"HTTPS"| BackendWorker["Cloudflare Worker (Backend API)"]
    AI["AI Clients (Claude / Cursor)"] -->|"MCP (SSE/HTTP)"| MCPWorker["Cloudflare Worker (MCP Server)"]
    MCPWorker -->|"HTTPS (Bearer Auth)"| BackendWorker
    BackendWorker -->|"SQL"| D1[("Cloudflare D1 (Database)")]
```

---

## 3. Component Breakdown

### 3.1 Web Dashboard (`brain_frontend`)
* **Technology:** React 19, TypeScript, Vite.
* **Styling System:** Modern Vanilla CSS design system (zero Tailwind).
  * Obsidian dark canvas (`#07080a`) with electric vermilion accents (`#ff4624`).
  * Editorial Japanese paper light mode (`[data-theme="light"]`).
  * Dot-grid blueprint canvas, levitating UI widgets, and Outcrowd-inspired split-screen narrative showcase.
* **Network Communication:**
  * Uses browser `fetch` with `credentials: 'include'` on all calls to `/api_brain/*`.
  * Zero client-side token storage in `localStorage` or `sessionStorage` (XSS-proof).
  * Initial boot calls `GET /api_brain/auth/me` to determine whether to render the dashboard or the authentication gateway.

### 3.2 Conversational AI Layer (`brain_mcp`)
* **Technology:** Cloudflare Workers, TypeScript, Model Context Protocol (MCP) TypeScript SDK.
* **Execution Environment:** Runs as a serverless edge Cloudflare Worker, exposing remote MCP transport over SSE / HTTP.
* **Communication:**
  * AI clients (Claude Desktop, Cursor, ChatGPT) connect remotely over the network.
  * Translates tool calls (`create_task`, `list_tasks`, `toggle_task`, `delete_task`) into authenticated HTTPS calls with `Authorization: Bearer <BRAIN_API_KEY>` to the Backend Worker.
  * Relays structured results back to the AI model's context stream.

### 3.3 Edge API Worker (`brain_backend`)
* **Technology:** Cloudflare Workers, Hono web framework (`^4.x`), TypeScript.
* **Domain & Base Path:** `https://dev.mohakapoor.in/api_brain` (production: `https://brain.mohakapoor.in/api_brain`).
* **Key Modules:**
  * **CORS Guard (`middleware/cors.ts`):** Allows only approved origins (`*.mohakapoor.in`, `localhost:5173`) with credentials enabled.
  * **Input Validation (`schemas/*`):** Zod 4 schemas validate and sanitize all JSON request payloads before controller execution.
  * **Dual Authentication (`middleware/auth.ts`):** Transparently verifies session cookies (Web) or Bearer tokens (MCP/CLI) and injects the authenticated `user` into the request context.
  * **Domain Controllers:**
    * `routes/auth.ts`: Registration (invite-guarded), login, logout, and `/me`.
    * `routes/tasks.ts`: Task CRUD with incomplete-first ordering.

### 3.4 Relational Storage (`Cloudflare D1`)
* **Engine:** Serverless SQLite at Cloudflare's edge.
* **Schema Integrity:** Foreign keys enforce `ON DELETE CASCADE`. If a user is deleted, all their sessions, API keys, and tasks are wiped automatically.
* **Epoch Timestamps:** All dates are stored as standard 64-bit Unix timestamps (`unixepoch()`) for minimal storage overhead and high-speed indexing.

---

## 4. Authentication & Security Architecture

```mermaid
sequenceDiagram
    autonumber
    actor User as Web Browser
    actor AI as MCP AI Agent
    participant Edge as Edge Worker (requireAuth)
    participant D1 as D1 SQLite Database

    Note over User,D1: Scenario A: Web Browser Request
    User->>Edge: GET /api_brain/tasks (Cookie: session_id=uuid)
    Edge->>D1: SELECT user FROM sessions WHERE id = ? AND expires_at > unixepoch()
    alt Session Valid
        D1-->>Edge: User Record (id, username)
        Edge-->>User: 200 OK + Tasks Payload
    else Session Missing or Expired
        Edge-->>User: 401 Unauthorized
    end

    Note over AI,D1: Scenario B: Headless MCP Agent Request
    AI->>Edge: POST /api_brain/tasks (Authorization: Bearer brain_live_xyz)
    Edge->>Edge: Compute SHA-256(brain_live_xyz)
    Edge->>D1: SELECT user FROM api_keys WHERE key_hash = ? AND (expires_at IS NULL OR expires_at > unixepoch())
    alt Key Valid
        D1-->>Edge: User Record (id, username)
        Edge-->>AI: 201 Created + New Task Payload
    else Key Invalid or Revoked
        Edge-->>AI: 401 Unauthorized
    end
```

### 4.1 Dual Authentication Matrix

| Attribute | Web Dashboard | MCP / AI Agent / CLI |
| :--- | :--- | :--- |
| **Transport** | `Cookie: session_id=<UUID>` | `Authorization: Bearer <API_KEY>` |
| **Storage in Browser/Client** | `HttpOnly`, `SameSite: Lax`, `Secure` | Local `.env` / config file |
| **Validation Mechanism** | Exact UUID lookup in `sessions` table | SHA-256 hash lookup in `api_keys` table |
| **Revocation** | Auto on `/logout` or single-device login override | Instant revocation via DB row deletion / toggle |
| **Session Lifetime** | 7 Days (`SESSION_TTL_SECONDS = 604800`) | Indefinite or configurable expiry |

### 4.2 Multi-Tenant Data Isolation
Every data mutation and selection is strictly scoped by user ID:
```sql
SELECT * FROM tasks WHERE user_id = ? ORDER BY completed ASC, created_at DESC;
UPDATE tasks SET completed = ? WHERE id = ? AND user_id = ?;
DELETE FROM tasks WHERE id = ? AND user_id = ?;
```
Even if an attacker guesses a valid `task_id`, the `WHERE user_id = ?` clause prevents unauthorized cross-tenant modification or deletion.

### 4.3 Invite-Only Registration Gate
To prevent unauthorized signups and bot account creation:
* `POST /api_brain/auth/register` requires `inviteCode`.
* Verified directly against the Cloudflare Worker secret `c.env.INVITE_CODE`.
* Fails fast with `403 Forbidden` if the code does not match.

---

## 5. End-to-End Data Flows

### 5.1 Web Dashboard: User Journey
1. **Bootstrap (`GET /auth/me`):**
   * App initializes. Calls `/auth/me` with `credentials: 'include'`.
   * If `200 OK`: user context is hydrated; renders `DashboardView`.
   * If `401 Unauthorized`: renders `AuthGateway` (Login/Register card).
2. **Task Creation (`POST /tasks`):**
   * User types into quick-capture input and hits `Enter`.
   * Payload `{ title: string }` validated by Zod schema (1–500 chars).
   * Edge worker inserts record into D1 with `completed = 0`.
   * Returns `201 Created` with full task row; UI prepends to list optimistically.
3. **Task Completion Toggle (`PATCH /tasks/:id`):**
   * Checkbox clicked. Dispatches `{ completed: true }`.
   * Server updates `completed = 1` and `completed_at = unixepoch()`.
   * Returns updated task; UI shifts item to the completed section with strikethrough transition.

### 5.2 MCP Conversational Agent: User Journey
1. **User Prompt:** "Claude, add 'Draft distributed systems memo' to my tasks."
2. **MCP Tool Selection:** Claude identifies the `create_task` tool with argument `{ title: "Draft distributed systems memo" }`.
3. **Execution:** Brain MCP server reads `BRAIN_API_KEY` from environment, performs `POST https://dev.mohakapoor.in/api_brain/tasks` with `Authorization: Bearer <key>`.
4. **Edge Processing:** Worker validates SHA-256 hash, attaches user, creates D1 row.
5. **Feedback Loop:** MCP server returns JSON to Claude; Claude informs the user: *"Added 'Draft distributed systems memo' to your tasks."*
6. **Instant Visibility:** Next time the user opens or refreshes the Web Dashboard, the new task is immediately present.

---

## 6. Infrastructure & Deployment Topology

| Component | Provider / Platform | Environment | Hostname |
| :--- | :--- | :--- | :--- |
| **Frontend SPA** | Vercel | Production | `brain.mohakapoor.in` |
| **Edge API Worker** | Cloudflare Workers | Dev / Staging | `dev.mohakapoor.in` |
| **Edge API Worker** | Cloudflare Workers | Production | `api.mohakapoor.in` / `brain.mohakapoor.in/api_brain` |
| **MCP Server Worker** | Cloudflare Workers | Dev / Prod | `mcp.mohakapoor.in` |
| **Database** | Cloudflare D1 | Isolated Dev | `brain-dev` |
| **Database** | Cloudflare D1 | Isolated Prod | `brain-prod` |
| **CI/CD** | GitHub Actions | Automated Pipeline | Lint, Typecheck, Migrations, Deploy |

---

## 7. Evolution & Roadmap Beyond Phase 1

1. **Phase 1 (Current):**
   * Core Todo & Task Foundation.
   * Remote D1 CRUD + Session Cookie Auth + Landing Showcase.
2. **Phase 2 (Immediate Next):**
   * Dual-Auth Middleware implementation (`sessions` + `api_keys`).
   * Cloudflare Worker `brain_mcp` server deployment (remote SSE/HTTP transport for Claude & Cursor).
   * Authenticated Web Dashboard views (Inbox, Today queue, Completed archive).
3. **Phase 3 (Expansion):**
   * Peripheral Life Modules (as previewed in [BeyondTasksShowcase.tsx](file:///c:/Users/mohak/OneDrive/Documents/Projects/brain/brain_frontend/src/components/BeyondTasksShowcase.tsx)):
     * Personal Finance & Burn Rate Telemetry.
     * Fitness & Kinetic Tonnage Tracking.
     * Knowledge Artifacts & Reference Library.
