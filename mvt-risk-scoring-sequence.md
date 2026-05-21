# Risk Scoring Pipeline — Classic MVT Pattern

> The "Fat View" problem: business logic, ORM orchestration, and side effects all crammed into the Django View layer.

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant View as View (risk_views.score_client)
    participant Model as Model (RiskScore / Client)
    participant DB as Database (Postgres)
    participant Tel as Telegram API
    participant Resp as Template/Response (JSON/HTML)

    rect rgb(232, 245, 233)
        Note over View: ⚠ FAT VIEW ZONE (Django MVT)

        Client->>+View: POST /clients/{id}/score/
        
        Note over View: Implicit ORM Fetch
        View->>+Model: Client.objects.get(id=id)
        Model->>+DB: SELECT * FROM clients WHERE id=...
        DB-->>-Model: client data
        Model-->>-View: client instance

        View->>+Model: Payment.objects.filter(client=client)
        Model->>+DB: SELECT * FROM payments WHERE client_id=...
        DB-->>-Model: payment rows
        Model-->>-View: payment queryset

        Note over View: ⚠ BUSINESS LOGIC IN VIEW
        View->>View: calculate_score_logic(client, payments)

        View->>+Model: RiskScore.objects.create(...)
        Model->>+DB: INSERT INTO risk_scoring_logs
        DB-->>-Model: saved
        Model-->>-View: log instance

        Note over View,Tel: ⚠ SIDE EFFECT IN VIEW
        View->>+Tel: telegram_bot.send_message(...)
        Tel-->>-View: ok

        View->>+Model: Notification.objects.create(...)
        Model->>+DB: INSERT INTO notifications
        DB-->>-Model: saved
        Model-->>-View: saved

        View->>+Resp: JsonResponse({ status, score })
        Resp-->>-Client: 200 OK
        deactivate View
    end
```

## Key Observations (The Django Context)

| # | Problem | Impact |
|---|---------|--------|
| 1 | **Business Logic in View** — The "V" in MVT acts as the orchestrator for weights, thresholds, and math. | The View becomes untestable without a mocked `HttpRequest`. Logic cannot be shared with `management commands`. |
| 2 | **ORM Over-Reliance** — The view micromanages multiple `filter()` and `get()` calls. | Leads to "N+1" query issues and makes the view tightly coupled to the database schema. |
| 3 | **Side Effects in View** — Dispatching Telegram messages or external API calls inside the request-response cycle. | A slow external API (Telegram) blocks the entire worker process, leading to timeouts for the user. |
| 4 | **Request Coupling** — Logic is often tied to `request.user` or `request.POST`. | Makes it impossible to trigger the same risk scoring from a background Celery task or an admin action. |

## Solution: The Service Layer (Current Implementation)

In our FastAPI codebase, we avoid the **Fat View** (MVT) or **Fat Controller** (MVC) by using a dedicated **Service Layer**.

```mermaid
flowchart TD
    Router[FastAPI Router / Django View] --> Service[RiskScoringService]
    Worker[Celery Worker] --> Service
    Command[CLI / Mgmt Command] --> Service
    
    Service --> Features[FeatureEngineeringService]
    Service --> Strategy[RiskScoringStrategy]
    Service --> DB[(Database Queries)]
```

By extracting the orchestration into `RiskScoringService`:
1. **Thin Entry Points**: Routers and Workers are just "callers" of the service.
2. **Isolated Logic**: The scoring strategy is a pure Python object, easily testable.
3. **Decoupled DB**: Database interactions are isolated in the `db/queries/` layer.
4. **Reliability**: Side effects (Telegram) are handled by a `NotificationService` or backgrounded immediately.