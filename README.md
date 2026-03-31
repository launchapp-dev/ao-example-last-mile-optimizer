# Last-Mile Delivery Optimizer

A complete last-mile delivery operations pipeline that clusters deliveries by geography, optimizes stop sequences for each driver, assigns routes based on vehicle capacity and driver familiarity, tracks delivery execution, resolves failed-delivery exceptions, and generates daily performance analytics — all fully automated with AO.

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DAILY DELIVERY OPERATIONS                        │
└─────────────────────────────────────────────────────────────────────┘

 delivery-orders.json
        │
        ▼
 ┌─────────────┐    rework (max 2)
 │  validate-  │◄──────────────────┐
 │   orders    │                   │
 │ (Haiku)     ├── rework ─────────┘
 └──────┬──────┘
        │ advance
        │ fail ──────────────────► [abort-operations] → abort-notice.md
        ▼
 ┌─────────────┐
 │   cluster-  │
 │ deliveries  │
 │  (Sonnet)   │
 └──────┬──────┘
        │
        ▼
 ┌─────────────┐    reroute (max 3)
 │  optimize-  │◄──────────────────┐
 │   routes    │                   │
 │  (Sonnet)   ├── reroute ────────┘
 └──────┬──────┘
        │ optimal / acceptable
        ▼
 ┌─────────────┐
 │   assign-   │──────► route-manifests.md
 │   drivers   │
 │  (Sonnet)   │
 └──────┬──────┘
        │
        ▼
 ┌─────────────┐    rework (max 5)
 │   track-    │◄──────────────────┐
 │  progress   │                   │
 │  (Sonnet)   ├── rework ─────────┘
 └──────┬──────┘
        │ exceptions
        │ completed ──────────────────────────────────────┐
        ▼                                                 │
 ┌─────────────┐                                         │
 │  handle-    │──────► exception-log.md                 │
 │ exceptions  │                                         │
 │  (Sonnet)   │                                         │
 └──────┬──────┘                                         │
        │ resolved / escalate                            │
        ▼                                                │
 ┌─────────────────────────────────────────────────────┐◄┘
 │               generate-analytics                    │
 │                    (Haiku)                          │
 └─────────────────────────────────────────────────────┘
        │
        ▼
   daily-dashboard.md + daily-analytics.json
```

## Quick Start

```bash
# 1. Navigate to the example directory
cd examples/last-mile-optimizer

# 2. Start the AO daemon
ao daemon start --autonomous

# 3. Enqueue a delivery batch
ao queue enqueue \
  --title "last-mile-optimizer" \
  --description "Process 2024-03-15 delivery batch — 35 orders, Austin TX metro" \
  --workflow-ref delivery-operations

# 4. Watch it run
ao daemon stream --pretty

# 5. Check outputs
cat documents/daily-dashboard.md
cat documents/route-manifests.md
```

Alternatively, let the **scheduled runs** handle it automatically:
- `5:00 AM Mon-Sat` — morning route planning (validates, clusters, optimizes, assigns)
- `12:00 PM Mon-Sat` — midday re-optimization (full pipeline on remaining stops)
- `7:00 PM Mon-Sat` — evening analytics (generates end-of-day performance report)

## Agents & Roles

| Agent | Model | Role |
|---|---|---|
| `order-intake` | claude-haiku-4-5 | Validates delivery batch, normalizes addresses, flags issues |
| `cluster-analyzer` | claude-sonnet-4-6 | Groups stops into geographic clusters (15-25 stops, 4-6 hrs each) |
| `route-optimizer` | claude-sonnet-4-6 | Sequences stops for minimum drive time, respects time windows |
| `driver-assigner` | claude-sonnet-4-6 | Matches routes to drivers by capacity, familiarity, proximity |
| `delivery-tracker` | claude-sonnet-4-6 | Monitors execution, flags exceptions (not home, wrong address, etc.) |
| `exception-handler` | claude-sonnet-4-6 | Resolves failures: reattempt, redirect, reschedule, return to depot |
| `ops-analyst` | claude-haiku-4-5 | Driver scorecards, route efficiency metrics, daily dashboard |

## Data Flow

| File | Written By | Read By |
|---|---|---|
| `data/delivery-orders.json` | (seed input) | order-intake |
| `data/driver-fleet.json` | (seed input) | driver-assigner |
| `data/service-area.json` | (seed input) | cluster-analyzer, route-optimizer |
| `data/validated-orders.json` | order-intake | cluster-analyzer |
| `data/delivery-clusters.json` | cluster-analyzer | route-optimizer |
| `data/optimized-routes.json` | route-optimizer | driver-assigner |
| `data/driver-assignments.json` | driver-assigner | delivery-tracker, exception-handler |
| `data/delivery-progress.json` | delivery-tracker | exception-handler, ops-analyst |
| `data/failed-deliveries.json` | exception-handler | ops-analyst |
| `data/daily-analytics.json` | ops-analyst | (final output) |
| `documents/route-manifests.md` | driver-assigner | (printable driver sheets) |
| `documents/exception-log.md` | exception-handler | (ops review) |
| `documents/daily-dashboard.md` | ops-analyst | (management report) |

## Key Decision Points

### 1. Validate Orders — `advance | rework | fail`
- **advance**: ≥70% of orders pass validation → proceed to clustering
- **rework**: 30-70% have fixable issues → re-validate batch (max 2 retries)
- **fail**: batch unprocessable → write abort notice and halt

### 2. Optimize Routes — `optimal | acceptable | reroute`
- **optimal**: all routes ≤9 hrs, efficiency ≥75 → assign drivers
- **acceptable**: routes feasible with minor overtime or efficiency dips → assign drivers
- **reroute**: routes unserviceable → send back to clustering (max 3 retries)

### 3. Track Progress — `completed | exceptions | rework`
- **completed**: all stops resolved → generate analytics
- **exceptions**: failed stops exist → run exception handler, then resume tracking
- **rework**: routes still active → advance simulation one step (max 5)

### 4. Handle Exceptions — `resolved | escalate`
- **resolved**: every failed stop has a disposition → continue to analytics
- **escalate**: damage claim >$500, fraud, or safety incident → analytics still runs, manager flagged

## Customizing

**Add a new driver:** Edit `data/driver-fleet.json` and add a driver entry with vehicle specs, start location, shift hours, and zone familiarity.

**Expand the service area:** Edit `data/service-area.json` — add new zones with zip codes and drive speed profiles, add package locker locations.

**Change clustering targets:** Edit the `cluster-analyzer` system prompt in `.ao/workflows/agents.yaml` — adjust the 15-25 stops target and 4-6 hour duration window.

**Change schedules:** Edit `.ao/workflows/schedules.yaml` — the cron expressions use standard 5-field syntax.

**Run only planning (no execution):** Use the `route-planning-only` workflow:
```bash
ao queue enqueue --title "last-mile-optimizer" --workflow-ref route-planning-only
```

**Run only analytics:** Use the `analytics-only` workflow after routes complete:
```bash
ao queue enqueue --title "last-mile-optimizer" --workflow-ref analytics-only
```

## AO Features Demonstrated

- **Multi-agent pipeline** — 7 specialized agents across 2 model tiers (Haiku + Sonnet)
- **Decision contracts** — structured verdicts (`advance/rework/fail`, `optimal/acceptable/reroute`, etc.)
- **Phase routing / rework loops** — clustering feeds back from route-optimizer; tracking loops with exceptions
- **Multi-path routing** — exceptions branch to handler then resume main pipeline
- **Command phase** — lightweight abort handler without a full agent
- **Scheduled workflows** — 3 daily cron jobs (5 AM planning, noon re-opt, 7 PM analytics)
- **Multiple workflow variants** — `delivery-operations`, `route-planning-only`, `analytics-only`
- **Output contracts** — structured JSON at each phase for downstream consumption

## Requirements

### MCP Servers (auto-installed via npx)
- `@modelcontextprotocol/server-filesystem` — read/write all data and document files
- `@modelcontextprotocol/server-sequential-thinking` — structured reasoning for clustering and route optimization

### No External API Keys Required
This example uses only reasoning and file I/O — no mapping APIs, no geocoding services, no external integrations. Everything runs offline with the seed data provided.

### Optional Integrations (not included)
For production use, you'd extend this with:
- **Google Maps Platform** or **OpenRouteService** for real geocoding and turn-by-turn directions
- **Twilio** for customer SMS notifications (delivery on way, failed attempt, redirect to locker)
- **Slack** via `@modelcontextprotocol/server-slack` for driver exception alerts to dispatch

## Domain Glossary

| Term | Definition |
|---|---|
| Last-mile delivery | The final leg of shipping — from distribution center to customer's door |
| Dwell time | Time spent at a stop (parking, walking, delivery attempt, signature) |
| First-attempt delivery rate | % of packages delivered successfully on the first try (industry target: 85%) |
| Time window | Customer's preferred delivery window (morning 8-12, afternoon 12-5, anytime) |
| Route manifest | Driver's printed stop-by-stop instructions with package details |
| Cluster | A geographic group of stops assigned to one driver's route |
| Deadhead miles | Miles driven without packages (depot→first stop, last stop→depot) |
| Exception | A delivery attempt that could not be completed as planned |
