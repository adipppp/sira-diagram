# Architecture Comparison: Traditional Monolith vs Decoupled Architecture

## Diagram 1 — Traditional Monolith (Django MVT)

> All backend participants (URLConf, View, ORM/Model, Template Engine) run inside **one process**.
> They share memory, state, and the same deployment lifecycle. The dashed line is logical, not physical.

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant URLConf as URLConf
    participant View as View (Business Logic + Orchestration)
    participant ORM as ORM / Model (Data Access)
    participant Template as Template Engine (HTML Rendering)
    participant DB as Database (Postgres)

    rect rgb(255, 235, 238)
        Note over URLConf,Template: ⚠ ONE PROCESS — Logical separation only
        Note over URLConf,Template: Deployed as a single unit on one server

        Client->>+URLConf: GET /invoices/overdue

        URLConf->>+View: route matched → call view function
        deactivate URLConf

        Note over View: Business Logic lives here<br>(calculations, filtering, permissions)

        View->>+ORM: Invoice.objects.filter(status='OVERDUE')

        ORM->>+DB: SELECT * FROM invoices WHERE status = 'OVERDUE'
        DB-->>-ORM: raw rows

        ORM-->>-View: QuerySet / model instances

        Note over View: Orchestrates: collects data,<br>prepares template context

        View->>+Template: render('overdue_list.html', context)

        Template-->>-View: Complete HTML page

        View-->>-Client: 200 OK — HTML

        Note over URLConf,Template: Everything inside this box shares:<br>• One Python process<br>• Same memory space<br>• Same deployment unit<br>• Same failure domain
    end
```

### What this means in practice

| Characteristic | Why it matters |
|---|---|
| **All logic in View** | The View owns business rules, DB queries, template preparation, and response construction — it's the "Fat Controller" |
| **Server-rendered HTML** | Every click requires a full round-trip; the server does all the presentation work |
| **Single process coupling** | A memory leak in the ORM crashes the template engine; a slow DB query blocks HTML rendering for all users |
| **Deployment** | Changing one line in a template requires redeploying the entire application |

---

## Diagram 2 — Decoupled Architecture (Your Project)

> Each participant runs in a **different process**, separated by network boundaries.
> Within the backend process, code is further organized into three logical layers with single responsibilities.

```mermaid
sequenceDiagram
    participant Client as Client (Browser — React SPA)
    participant Router as Router (HTTP Concerns)
    participant Service as Service Layer (Business Logic)
    participant DBQueries as DB Queries (Data Access)
    participant DB as Database (Postgres / Supabase Cloud)

    rect rgb(232, 245, 233)
        Note over Client: Presentation Tier
        Note over Client: Runs in browser — separate process, separate deployment

        Client->>+Router: GET /api/invoices/overdue
        Note over Client,Router: ═══ NETWORK BOUNDARY (HTTP) ═══

        rect rgb(227, 242, 253)
            Note over Router,DBQueries: Application Tier (FastAPI in Docker)
            Note over Router,DBQueries: One process, but code is separated into layers with distinct responsibilities

            Note over Router: Thin: validates query params,<br>delegates to service, maps errors

            Router->>+Service: listOverdue(filters)

            Note over Service: Business Logic:<br>• Calculates days_overdue<br>• Hydrates client fields<br>• Applies business rules

            Service->>+DBQueries: getOverdueInvoices(db, filters)

            Note over DBQueries: Pure Data Access:<br>• Builds SQL query<br>• No business rules<br>• Returns raw dicts

            DBQueries->>+DB: SELECT ... FROM invoices WHERE status = 'OVERDUE'
            Note over DBQueries,DB: ═══ NETWORK BOUNDARY (Postgres wire protocol) ═══

            rect rgb(255, 243, 224)
                Note over DB: Data Tier (Supabase Cloud — separate infrastructure)
            end

            DB-->>-DBQueries: rows

            DBQueries-->>-Service: list[dict]

            Service-->>-Router: InvoiceResponse (enriched)

            Router-->>-Client: 200 OK — JSON
            Note over Router,Client: ═══ NETWORK BOUNDARY (HTTP) ═══
        end
    end
```

### What this means in practice

| Characteristic | Why it matters |
|---|---|
| **Business Logic in Service Layer** | Router doesn't compute — it delegates. Service is pure Python, testable without HTTP |
| **JSON response, not HTML** | The client decides how to render the data. React, mobile app, or CLI — same API |
| **Network boundaries** | Each tier can fail, scale, and deploy independently. A DB slowdown doesn't block the React app from loading |
| **Deployment** | Frontend can be deployed to a CDN; backend to Docker; DB managed by Supabase — all independent |

---

## Side-by-Side Comparison

| Dimension | Monolith (Django MVT) | Your Architecture |
|---|---|---|
| **Process structure** | One process for all backend logic | Three tiers (Browser → Docker → Cloud DB) |
| **Deployment unit** | One (the entire app) | Three independent units |
| **Code organization** | Logic in View / Model | Router → Service → DB Queries |
| **Request output** | HTML (server-rendered) | JSON (client-rendered) |
| **Business logic location** | Scattered across View and Model | Centralized in Service Layer |
| **Database access** | ORM inline in View / Model | Isolated in DB Queries layer |
| **Testability** | Need HTTP client to exercise logic | Service tested with pure Python calls |
| **Scalability** | Vertical — scale the whole monolith | Horizontal per tier |
| **Failure isolation** | Any exception can crash the process | One tier can degrade without taking down others |
| **Frontend flexibility** | Tied to server templates | Any client can consume the API |
