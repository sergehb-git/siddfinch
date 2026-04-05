# Plan: League manager web app (Python, Docker, Render)

**Goal:** A Python web application, **containerized with Docker**, deployed on **Render**, that can eventually ingest **Rotowire league data** (read-only) and surface **recommendations** (lineup, categories, FA targets) aligned with **2026 NL league rules** (`2026/rules/2026-rules.md`).

**Primary audience:** **You and every other league manager** (e.g. **11 teams** in 2026). Each manager is expected to **sign in** and use the app for **their** roster, planning, and recommendations.

**Non-users:** The **league secretary** does **not** use this application. The secretary continues to execute roster changes on **Rotowire** and to coordinate moves from **email** (or other league process). Managers may use the app to **draft emails** or **track requests** sent to the secretary—**outbound only** from the app’s perspective.

**Requirement:** **Multi-tenant by design** from the first shippable version: **authentication**, **per-manager data isolation**, and a clear binding between **login identity** and **league team** (e.g. Shatners, Kiev Ghosts).

**Design standard:** Implement and review code against **John Ousterhout**, *A Philosophy of Software Design*—see project rule **`.cursor/rules/philosophy-of-software-design.mdc`**.

**Constraints / context:**

- **Managers** use this app; **Rotowire** remains the league host (read automation **optional** / **brittle**). **Secretary** updates Rotowire outside the app.
- Render **free web services** **spin down** when idle; first request after sleep pays a **cold start**. Plan for that in UX (loading state) or upgrade to always-on later.

---

## Objectives (phased)

| Phase | Outcome |
| --- | --- |
| **0 — Scaffold** | Repo layout, `Dockerfile`, health endpoint, `PORT` binding, local `docker compose` optional. |
| **1 — Deploy** | Git-connected **Render** deploy from Dockerfile; secrets via Render **environment**; smoke test URL. |
| **2 — Auth (managers)** | **Every manager** can have an account: register/login (or **invite-only**), **session cookies** or **JWT**, passwords hashed (**bcrypt** / **argon2**). Each account is linked to exactly one **league team** (enforced or admin-assigned). No shared “league password.” |
| **3 — Data model** | Versioned **league snapshot** and all private artifacts scoped by **`user_id` / `team_profile_id`** (required on write). League-wide **read** data (e.g. full standings) can be stored once per snapshot **version** with shared read access rules, or duplicated per tenant—pick one approach and document invariants. |
| **4 — Ingest v1** | **Manual import** (paste JSON/CSV or upload) **per logged-in manager** so the app works **without** Rotowire login. |
| **5 — Brain v1** | Rule-based **recommendations** from snapshot + `2026-rules.md` categories (offense: AVG, R, RBI, SB, TB+BB+HBP; pitching: W, SV, K, ERA, QS). |
| **6 — Rotowire connector (optional)** | Playwright or session-based **read-only** fetch; **per-user Rotowire session** if each manager uses their own Rotowire login, or a single service account if league policy allows—document security implications. |
| **7 — Email helpers (optional)** | **Managers** generate **outbound** messages (e.g. FA bids, add/drop requests) to the **secretary’s** address; optional **inbound** parse of **replies** to update “request status” in-app. The secretary never logs into this app. |

---

## Users and authentication (strategy)

- **Who signs in:** **League managers only** (you + other owners). **Not** the secretary.
- **Accounts:** `email`, `password_hash`, `display_name`; optional **OAuth** (Google/GitHub) later—still map to **`user_id`** and **team**.
- **Team binding:** One **team profile** per manager (e.g. “Shatners”). Admins or **invite tokens** assign `team_profile_id` so managers cannot claim each other’s team.
- **Authorization:** All **private** rows (imports, notes, FA targets, email drafts, per-team snapshot copies) require **`user_id`** match (or role-based admin for league-wide ops).
- **League-wide visibility:** Define explicitly what **all** managers may see (e.g. aggregate standings) vs **owner-only** (bids, watchlists). Prefer **one** policy module so rules don’t sprawl (Ousterhout: general mechanism).
- **Local dev:** Use a **seed script** with 2–3 test accounts representing different teams; production uses real manager emails.

---

## Technical choices (defaults)

- **Framework:** **FastAPI** + **Uvicorn** (async-friendly, OpenAPI docs, easy JSON APIs).
- **Container:** Single-stage or slim `python:3.12-slim` image; non-root user optional hardening pass.
- **Render:** **Web Service** from **Dockerfile**; set `PORT` (Render injects); command listens on `0.0.0.0`.
- **Persistence:** Start with **SQLite** in a **Render disk** (if attached) or switch early to **Render Postgres** if you need multi-instance or reliable file storage (ephemeral filesystem on free tier is a gotcha—**Postgres is safer** for production data).

---

## Software design (Ousterhout)

Apply *A Philosophy of Software Design* to this codebase—not as ceremony, but to **keep complexity from compounding**.

| Principle | How it shows up here |
| --- | --- |
| **Strategic > tactical** | Refactor when a feature would scatter special cases; don’t stack `if league == …` across layers. |
| **Deep modules** | **Narrow public API**, rich internals: e.g. one **`LeagueSnapshot`** type + **`SnapshotStore`** (load/save/version); **`RotowireReader`** (optional) hides all HTML/session mess behind `fetch_snapshot() → LeagueSnapshot` or a clear failure. |
| **Pull complexity down** | Parsing Rotowire, normalizing player IDs, and NL-only rules belong **inside** ingest/domain—not in route handlers. |
| **Different layer, different abstraction** | HTTP layer maps requests/responses only; **no pass-through** services that merely re-export the DB. |
| **Avoid temporal decomposition** | Package by **knowledge**, not pipeline step names: `domain/`, `ingest/`, `recommendations/`, `persistence/`, `api/` (or equivalent)—not `step1_load`, `step2_parse`. |
| **Information hiding** | Persist snapshots as a **versioned blob + schema version** if it reduces coupling; expose **invariants** (“snapshot is immutable per id”) in module docstrings. |
| **General mechanisms** | One **import pipeline** for manual JSON and Rotowire output; avoid duplicate validators per source. |
| **Define errors out** | Prefer validation at import boundaries so recommendation code assumes a **valid** snapshot. |
| **Comments at module level** | File/class docstrings for **why**, tradeoffs, and invariants; avoid line-by-line narration. |
| **Design it twice** | For ingest + auth boundaries, briefly sketch **two** shapes (e.g. fat adapter vs thin ORM) and pick the **lower long-term complexity** option. |

### Module layering (target dependencies)

```mermaid
flowchart TB
  API[api - HTTP only]
  REC[recommendations]
  DOM[domain - types & invariants]
  ING[ingest]
  PER[persistence]
  AUTH[auth - managers]

  API --> REC
  API --> ING
  API --> PER
  API --> AUTH
  REC --> DOM
  ING --> DOM
  PER --> DOM
  AUTH -.->|user id for scoping| PER
```

Lower layers (`domain`) must not import `api` or FastAPI.

---

## Repository layout (target)

Organize by **domain responsibility** (Ousterhout: avoid temporal decomposition). Adjust names to taste; keep **seams** clear.

```
app/
  main.py              # app factory, mount routers only
  api/                 # HTTP: thin handlers, DTOs, deps
  domain/              # LeagueSnapshot, team, player, category keys—no FastAPI
  ingest/              # manual import, future Rotowire → domain types
  recommendations/     # pure functions / service over snapshot + rules
  persistence/         # store/load snapshots, users, team profiles—hide SQL details
  auth/                # login, sessions, team binding—required for production
data/                  # sample snapshots for dev/tests
tests/
Dockerfile
.dockerignore
requirements.txt
render.yaml            # optional: Render IaC
plans/                 # this document
.cursor/rules/         # philosophy-of-software-design.mdc (Ousterhout)
```

---

## Render + Docker checklist

1. **Dockerfile** `CMD` uses `$PORT` (e.g. `sh -c 'uvicorn ... --port ${PORT:-8000}'`).
2. **Health check** route (`/health`) for Render **health checks**.
3. **`.dockerignore`** excludes `.git`, `.venv`, `__pycache__`.
4. **Environment variables** on Render: no secrets in image; use dashboard or `render.yaml`.
5. **Cold starts:** document that first load after idle may lag; avoid long synchronous scrapes on the **first** request (use background job or manual trigger).

---

## Security notes

- **Rotowire credentials** (if ever used): env vars only; never commit; rotate if logs leak.
- **Rate limiting** on auth and import endpoints; **account enumeration** resistance on login/register.
- Prefer **read-only** automation; align with Rotowire **terms of use**.
- **Managers:** Never store plaintext passwords; **HTTPS** on Render (default); **invite-only** or **email verification** recommended so random signups cannot claim a team.

---

## Workflow diagrams

### End-to-end system context

```mermaid
flowchart TB
  subgraph managers [League managers]
    M1[You + other owners]
  end

  subgraph render [Render]
    W[Web Service - Docker]
    A[Login sessions per manager]
  end

  subgraph data [Persistence]
    DB[(Postgres or SQLite)]
  end

  subgraph secretary [Secretary - outside app]
    SEC[Email inbox]
    RWwork[Rotowire admin]
  end

  subgraph rotowire [Rotowire]
    RW[League pages - read for managers]
  end

  M1 -->|HTTPS + auth| A
  A --> W
  W --> DB
  W -.->|optional read-only sync| RW
  M1 -->|transaction emails| SEC
  SEC --> RWwork
  RWwork -->|updates rosters| RW
```

### Request / data flow inside the app

```mermaid
flowchart LR
  subgraph ingest [Ingestion]
    M[Manual import API or UI]
    P[Rotowire connector - optional]
  end

  subgraph core [Application core]
    V[Validate + normalize snapshot]
    S[Store snapshot version]
    R[Recommendation engine]
  end

  subgraph out [Output]
    API[JSON / dashboard]
  end

  M --> V
  P --> V
  V --> S
  S --> R
  R --> API
```

### Deploy loop (Git → Render)

```mermaid
flowchart LR
  DEV[Local dev - Docker Desktop] --> GIT[Git push]
  GIT --> GH[GitHub]
  GH --> RD[Render build + deploy]
  RD --> LIVE[Public URL]
  LIVE --> DEV
```

---

## Success criteria

- **Docker:** `docker build` and `docker run -p 8000:8000` serve a healthy app locally.
- **Render:** Auto-deploy on push; `/health` returns 200; no hard-coded secrets.
- **Product:** With a **mock or pasted** league snapshot, the UI or API returns at least one **actionable recommendation** (e.g. weakest category vs league median).
- **Multi-manager:** **Two or more** test accounts can log in concurrently; each sees **only** their team’s private data; shared league views behave per policy.
- **Design:** New features **default** to new code behind **existing module boundaries**; route files stay thin; **no duplicated** league-rule logic outside `domain` / `recommendations`.

---

## Open decisions (fill in as you go)

- **Roto vs points** scoring on Rotowire (affects recommendation math).
- **Postgres vs SQLite** on Render (filesystem persistence vs managed DB).
- **Registration policy:** **invite-only** (commish/seeds accounts) vs **open sign-up** with manual team approval.
- **League-wide data:** single shared snapshot vs per-tenant copy—impacts storage and consistency.

---

*Last updated: all managers are users; secretary excluded from app; Ousterhout-aligned layout.*
