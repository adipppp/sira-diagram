# Risk Scoring Pipeline — Classic MVC Pattern

> The "Fat Controller" problem: business logic, external API calls, and DB orchestration all crammed into one layer.

```mermaid
sequenceDiagram
    participant Client as Client (Browser)
    participant Ctrl as Controller (RiskController)
    participant Model as Model (RiskModel)
    participant DB as Database (Postgres)
    participant Tel as Telegram API
    participant View as View (Response Render)

    rect rgb(255, 235, 238)
        Note over Ctrl: ⚠ FAT CONTROLLER ZONE

        Client->>+Ctrl: POST /api/clients/{id}/score

        Ctrl->>+Model: getClient(id)
        Model->>+DB: SELECT * FROM clients WHERE id=...
        DB-->>-Model: client data
        Model-->>-Ctrl: client data

        Ctrl->>+Model: getPaymentHistory(id)
        Model->>+DB: SELECT * FROM payments WHERE client_id=...
        DB-->>-Model: payment data
        Model-->>-Ctrl: payment data

        Note over Ctrl: ⚠ BUSINESS LOGIC IN CONTROLLER
        Ctrl->>Ctrl: calculateRiskScore(client, payments)

        Ctrl->>+Model: saveScore(id, score, features)
        Model->>+DB: INSERT INTO risk_scoring_logs
        DB-->>-Model: saved
        Model-->>-Ctrl: saved

        Note over Ctrl,Tel: ⚠ EXTERNAL API CALL IN CONTROLLER
        Ctrl->>+Tel: sendTelegramAlert(company, risk)
        Tel-->>-Ctrl: delivered

        Ctrl->>+Model: createNotification(data)
        Model->>+DB: INSERT INTO notifications
        DB-->>-Model: done
        Model-->>-Ctrl: done

        Ctrl->>+View: render(scoreResponse)
        View-->>-Client: 200 OK — JSON { risk_label, score }
        deactivate Ctrl
    end
```

## Key Observations

| # | Problem | Impact |
|---|---------|--------|
| 1 | **Business Logic in Controller** — the risk scoring formula, weights, and thresholds are computed directly in the Controller | Cannot unit-test the formula without simulating an HTTP request |
| 2 | **External API Call in Controller** — Telegram notification dispatched directly | If Telegram is down, the HTTP request hangs |
| 3 | **Multiple DB Round-Trips** — Controller micromanages every database call | Violates Law of Demeter; brittle to schema changes |
| 4 | **No Reusability** — a Celery worker needs the same logic | Must copy-paste all of the above; no single `score_client()` to call |

## Solution: Service Layer

```mermaid
flowchart LR
    A[Router] --> B[RiskScoringService]
    B --> C[FeatureEngineeringService]
    B --> D[RiskScoringStrategy]
    D --> E[RuleBasedStrategy]
    D --> F[MLStrategy - future]
    B --> G[(Database)]
    B --> H[NotificationService]
    H --> I[Telegram]
    H --> J[In-App Notification]
```

The **Service Layer** extracts all orchestration into `RiskScoringService`, so:

1. Both the **Router** and the **Celery Worker** call `service.score_client(id)` — no duplication
2. The scoring formula is pluggable via the `RiskScoringStrategy` Protocol (Open/Closed Principle)
3. External side-effects (Telegram, notifications) are isolated in dedicated services
4. Pure logic, testable without HTTP context
