# Architecture Comparison: Traditional Monolith vs Decoupled Architecture

## Traditional Monolithic (MVC/MVT)

```mermaid
flowchart TD
    subgraph M1["Single Process (e.g. Django / Rails)"]
        direction TB
        
        subgraph MURL["URL Router"]
            URL[urls.py]
        end
        
        subgraph MVC["View / Controller Layer"]
            VC["View / Controller (contains business logic)"]
        end
        
        subgraph MMODEL["Model Layer"]
            MDEF[(Model / ORM and data access)]
        end
        
        subgraph MTMPL["Template Engine"]
            T["Template (HTML) server-side rendering"]
        end
        
        URL --> VC
        VC --> MDEF
        MDEF --> DB[(Database)]
        DB --> MDEF
        MDEF --> VC
        VC --> T
    end

    B1[Browser] -- "HTTP Request" --> URL
    T -- "Completed HTML Page" --> B1

    style B1 fill:#E8F5E9,stroke:#388E3C
    style VC fill:#FFEBEE,stroke:#D32F2F
    style T fill:#E8F5E9,stroke:#388E3C
    style DB fill:#FFF3E0,stroke:#E65100
    linkStyle 0,1,5,6 stroke:#D32F2F,stroke-width:2
```

**Key characteristics:**
- All code runs in **one process** — the browser receives pre-rendered HTML
- The View/Controller is a **"fat" layer** handling logic, auth, rendering
- The Template Engine is **tightly coupled** to the backend — HTML lives inside the server
- Scaling means scaling **everything** together (vertical scaling)

---

## Zoom In: Risk Scoring Pipeline inside the Monolith

```mermaid
flowchart TD
    subgraph FAT["Fat Controller - risks.py (ALL logic mixed together)"]
        direction TB
    
        A["validate_request()"]
        B["get_client_by_id(id)"]
        C["get_payments_by_client(id)"]
        D["calc_delay_score()"]
        E["calc_overdue_count()"]
        F["calc_outstanding_amt()"]
        G["calc_payment_consistency()"]
        H["calc_invoice_age()"]
        I["apply_weighted_formula()"]
        J["determine_risk_label()"]
        K["insert_risk_scoring_log()"]
        L["update_client_risk()"]
        M["if HIGH: send_telegram_alert()"]
        N["if HIGH: create_notification()"]
        O["render_json_response()"]
        
        A --> B --> C --> D --> E --> F --> G --> H --> I --> J
        J --> K --> L --> M
        M --> N --> O
        L --> O
    end
    
    DB[(Database)]
    EXT[Telegram API]
    
    B -.-> DB
    C -.-> DB
    K -.-> DB
    L -.-> DB
    M -.-> EXT
    
    style FAT fill:#FFEBEE,stroke:#D32F2F,stroke-width:3
    style A fill:#FFCDD2
    style B fill:#FFCDD2
    style C fill:#FFCDD2
    style D fill:#FFCDD2
    style E fill:#FFCDD2
    style F fill:#FFCDD2
    style G fill:#FFCDD2
    style H fill:#FFCDD2
    style I fill:#FFCDD2
    style J fill:#FFCDD2
    style K fill:#FFCDD2
    style L fill:#FFCDD2
    style M fill:#FFCDD2
    style N fill:#FFCDD2
    style O fill:#FFCDD2
    style DB fill:#FFF3E0
    style EXT fill:#F3E5F5
    
    linkStyle 14,15,16,17,18 stroke:#D32F2F,stroke-width:2,stroke-dasharray:3
```

**Every responsibility crammed into one layer:**
- Data access (queries mixed with logic) — steps B, C
- Feature calculation (5 separate metrics) — steps D-H
- Scoring formula and thresholds — steps I, J
- Persistence and updates — steps K, L
- External side effects (Telegram) — step M
- In-app notification logic — step N
- Response rendering — step O

None of these can be reused, tested in isolation, or swapped independently.

---

## Your Architecture (Decoupled / Three-Tier)

```mermaid
flowchart TD
    subgraph T1["Tier 1: Presentation"]
        REACT["React SPA (Vite) - apps/web/"]
    end

    subgraph T2["Tier 2: Application"]
        direction TB
        
        subgraph L1["Router Layer (Thin)"]
            R["FastAPI Router - routers/invoices.py"]
        end
        
        subgraph L2["Service Layer (Business Logic)"]
            S["Service Classes - services/invoice_service.py"]
        end
        
        subgraph L3["Strategy Layer (Pluggable)"]
            STRAT["Strategy Protocol - risk_service.py - RiskScoringStrategy"]
        end
        
        subgraph L4["DB Queries Layer"]
            Q["Query Functions - db/queries/invoices.py"]
        end
        
        R --> S
        S --> Q
        S --> STRAT
    end

    subgraph T3["Tier 3: Data"]
        DB[(Supabase / PostgreSQL - supabase/migrations/)]
    end

    subgraph BP["Background Processing"]
        CELERY["Celery Worker - workers/score_all_clients.py"]
    end

    REACT -- "JSON API calls" --> R
    Q --> DB
    CELERY --> S
    CELERY --> Q

    style REACT fill:#E8F5E9,stroke:#388E3C
    style R fill:#E3F2FD,stroke:#1565C0
    style S fill:#F3E5F5,stroke:#7B1FA2
    style STRAT fill:#FFF8E1,stroke:#F57F17
    style Q fill:#E0F2F1,stroke:#00695C
    style DB fill:#FFF3E0,stroke:#E65100
    style CELERY fill:#FCE4EC,stroke:#c62828
    linkStyle 0,1,2,3,4 stroke:#1565C0,stroke-width:2
```

**Key characteristics:**
- **Three physical tiers** — Presentation (browser), Application (Docker), Data (Supabase)
- **Router is thin** — owns only HTTP concerns (auth, status codes, validation)
- **Service Layer owns logic** — pure Python, testable without HTTP
- **Strategy Protocol** — scoring formula is pluggable (OCP/DIP via `RiskScoringStrategy`)
- **DB Queries are isolated** — data access lives in dedicated files, not mixed with logic
- **Background worker** — Celery independently calls the same Service Layer (pink node)
- Each component can be **scaled independently**

---

## Zoom In: Risk Scoring Pipeline in Your Architecture

```mermaid
flowchart TD
    subgraph ROUTER["Router Layer (thin)"]
        R["POST /api/clients/{id}/score"]
    end
    
    subgraph SERVICE["Service Layer"]
        S["RiskScoringService.score_client(id)"]
    end
    
    subgraph FE["Feature Engineering"]
        FE_SVC["FeatureEngineeringService.extract_features()"]
    end
    
    subgraph STRATEGY["Strategy (pluggable)"]
        STRAT["RiskScoringStrategy Protocol"]
        RB["RuleBasedScoringStrategy"]
        ML["MLScoringStrategy (future)"]
    end
    
    subgraph QUERIES["DB Queries Layer"]
        Q_CLIENT["queries/clients/get_client_by_id"]
        Q_SAVE["queries/risk_scoring/insert_risk_scoring_log"]
        Q_UPDATE["queries/risk_scoring/update_client_risk"]
    end
    
    subgraph SIDE["Side Effects"]
        TG["notify_high_risk_flagged() - Telegram"]
        NOTIF["NotificationService.create() - in-app"]
    end
    
    DB[(Database)]
    EXT[Telegram API]
    
    R --> S
    S --> FE_SVC
    S --> STRAT
    STRAT -.-> RB
    STRAT -.-> ML
    S --> Q_CLIENT
    S --> Q_SAVE
    S --> Q_UPDATE
    Q_CLIENT --> DB
    Q_SAVE --> DB
    Q_UPDATE --> DB
    S --> TG
    S --> NOTIF
    TG --> EXT
    NOTIF --> DB

    style R fill:#E3F2FD,stroke:#1565C0
    style S fill:#F3E5F5,stroke:#7B1FA2
    style FE_SVC fill:#FFF8E1,stroke:#F57F17
    style STRAT fill:#FFF8E1,stroke:#F57F17
    style RB fill:#FFF8E1,stroke:#F57F17
    style ML fill:#FFF8E1,stroke:#F57F17,stroke-dasharray:3
    style Q_CLIENT fill:#E0F2F1,stroke:#00695C
    style Q_SAVE fill:#E0F2F1,stroke:#00695C
    style Q_UPDATE fill:#E0F2F1,stroke:#00695C
    style TG fill:#FCE4EC,stroke:#c62828
    style NOTIF fill:#FCE4EC,stroke:#c62828
    style DB fill:#FFF3E0,stroke:#E65100
    style EXT fill:#F3E5F5,stroke:#7B1FA2

    linkStyle 0,1,2,3,4,5,6,7,8,9,10,11 stroke:#1565C0,stroke-width:2
```

**How it differs from the monolith:**

| Aspect | Monolith Controller | Your Architecture |
|--------|-------------------|-------------------|
| **Data access** | Mixed into controller | Isolated in `db/queries/` |
| **Scoring formula** | Inline in controller function | Pluggable `RiskScoringStrategy` Protocol |
| **Feature extraction** | Inline calculations | `FeatureEngineeringService` |
| **Side effects** (Telegram, notifications) | Hard-coded in controller | Dedicated `NotificationService` + `Telegram` |
| **Reusability** | Cannot reuse without HTTP | Celery worker calls same `score_client()` |
| **Testability** | Need HTTP request + DB mock | Pure async; inject mock strategy + real DB |

---

## At a Glance

| Aspect | Traditional Monolith (Django) | Your Architecture |
|---|---|---|
| **Deployment** | One server process | Browser + Docker + Supabase |
| **Frontend location** | Inside the backend (HTML templates) | Separate app (`apps/web`) |
| **Business logic lives in** | View / Controller (fat) | Service Layer |
| **Database access** | ORM mixed into models | Isolated query files |
| **Background jobs** | Same process or hacky cron | Dedicated Celery worker |
| **Scalability** | Vertical (scale the one server) | Horizontal (scale each tier independently) |
| **Testing logic** | Needs simulated HTTP request | Pure async unit tests |
| **Code organization** | Monorepo *and* monolith | Monorepo *but* decoupled modules |

---

## The Key Insight

> **Monorepo ≠ Monolith.** You share a Git repository for convenience, but your code runs as independent, loosely-coupled systems that communicate over HTTP. A true monolith runs as one process where the frontend, backend, and database access are all interleaved in shared memory.
