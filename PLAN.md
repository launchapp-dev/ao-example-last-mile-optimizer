# Last-Mile Delivery Optimizer — Build Plan

## Overview

A last-mile delivery operations pipeline that takes a batch of delivery orders, clusters them by geography, optimizes route sequences within each cluster, assigns drivers based on capacity and location, tracks delivery progress, handles failed deliveries (reattempt/redirect/return-to-depot), and generates daily analytics on driver performance and operational efficiency.

This example showcases AO as an **operations automation tool** — not coding, but real logistics coordination with geographic clustering, route sequencing, exception handling loops, and scheduled re-optimization.

---

## Agents

| Agent ID | Model | Role |
|---|---|---|
| `order-intake` | claude-haiku-4-5 | Validates incoming delivery orders batch, normalizes addresses, flags issues |
| `cluster-analyzer` | claude-sonnet-4-6 | Groups deliveries into geographic clusters using proximity/density analysis |
| `route-optimizer` | claude-sonnet-4-6 | Sequences stops within each cluster for minimum distance/time, respects time windows |
| `driver-assigner` | claude-sonnet-4-6 | Matches routes to available drivers by capacity, vehicle type, start location |
| `delivery-tracker` | claude-sonnet-4-6 | Monitors delivery progress, flags late/failed/skipped stops, updates ETAs |
| `exception-handler` | claude-sonnet-4-6 | Resolves failed deliveries — reattempt, redirect to neighbor/locker, return to depot |
| `ops-analyst` | claude-haiku-4-5 | Generates driver scorecards, route efficiency metrics, daily operations dashboard |

---

## Data Flow

```
data/delivery-orders.json           (input — batch of day's deliveries)
data/driver-fleet.json              (seed — available drivers, vehicles, start locations)
data/service-area.json              (seed — zones, boundaries, depot locations)
data/validated-orders.json          (written by order-intake)
data/delivery-clusters.json         (written by cluster-analyzer)
data/optimized-routes.json          (written by route-optimizer)
data/driver-assignments.json        (written by driver-assigner)
data/delivery-progress.json         (written by delivery-tracker)
data/failed-deliveries.json         (written by exception-handler)
data/daily-analytics.json           (written by ops-analyst)
documents/route-manifests.md        (written by driver-assigner — printable driver sheets)
documents/daily-dashboard.md        (written by ops-analyst — operations summary)
documents/exception-log.md          (written by exception-handler — failed delivery log)
```

---

## Phase Pipeline

### Phase 1: `validate-orders` (agent: order-intake)
- Reads `data/delivery-orders.json` (batch of 20-50 orders)
- Validates each order: address present, valid time window, package dimensions/weight within limits
- Normalizes addresses, geocodes approximate lat/lng from zip codes
- Flags orders with issues (missing apartment number, business closed on delivery day, oversized)
- Writes `data/validated-orders.json`
- **Verdict:** `advance` (valid batch) | `rework` (>30% orders have issues) | `fail` (no valid orders)

### Phase 2: `cluster-deliveries` (agent: cluster-analyzer)
- Reads `data/validated-orders.json` and `data/service-area.json`
- Groups deliveries into geographic clusters (target: 15-25 stops per cluster)
- Uses proximity analysis: zip code grouping, zone assignment, density-based splitting
- Balances cluster sizes by package volume and time window constraints
- Each cluster maps to one driver route
- Writes `data/delivery-clusters.json`
- **Always advances** — clustering result is evaluated in route optimization

### Phase 3: `optimize-routes` (agent: route-optimizer)
- Reads `data/delivery-clusters.json` and `data/service-area.json`
- For each cluster, sequences stops to minimize total drive time
- Respects delivery time windows (morning-only, afternoon-only, anytime)
- Accounts for depot start/end, left-turn avoidance heuristics, residential vs commercial patterns
- Estimates per-stop timing: drive time + dwell time (2 min residential, 5 min commercial, 10 min bulk)
- Calculates total route distance, estimated duration, and feasibility
- Writes `data/optimized-routes.json`
- **Verdict:** `optimal` (all routes feasible within shift) | `acceptable` (feasible with overtime) | `reroute` (some routes exceed shift limits, need re-clustering)

### Phase 4: `assign-drivers` (agent: driver-assigner)
- Reads `data/optimized-routes.json` and `data/driver-fleet.json`
- Matches routes to drivers by:
  - Vehicle capacity vs route total volume/weight
  - Vehicle type (van for residential, truck for commercial/bulk)
  - Driver start location proximity to route's first stop
  - Driver shift availability and hours remaining
  - Driver zone familiarity (prefer drivers who know the area)
- Generates driver manifests with turn-by-turn stop order, package list per stop, time targets
- Writes `data/driver-assignments.json`
- Writes `documents/route-manifests.md`
- **Always advances** after assignment

### Phase 5: `track-progress` (agent: delivery-tracker)
- Reads `data/driver-assignments.json` for scheduled routes
- Reads `data/delivery-progress.json` (if exists) for current state
- Simulates delivery execution: marks stops as completed, in-progress, or failed
- Flags exceptions:
  - Customer not home (no safe place to leave package)
  - Wrong address / address not found
  - Package damaged in transit
  - Access denied (gated community, locked building)
  - Customer refused delivery
  - Running behind schedule (>30 min late to remaining stops)
- Updates ETAs for remaining stops
- Writes `data/delivery-progress.json`
- **Verdict:** `completed` (all routes finished) | `exceptions` (failed deliveries need handling) | `rework` (still in progress, simulate advancing)

### Phase 6: `handle-exceptions` (agent: exception-handler)
- Reads `data/delivery-progress.json` for failed deliveries
- Reads `data/driver-assignments.json` for route context
- For each failed delivery, determines action:
  - **Reattempt today** — customer now available, driver nearby, time permits
  - **Redirect** — leave at neighbor, deliver to nearby locker/access point
  - **Next-day reattempt** — schedule for tomorrow's batch
  - **Return to depot** — customer unreachable after 2 attempts, hold for pickup
- Updates delivery status and generates exception log
- Writes `data/failed-deliveries.json`
- Writes `documents/exception-log.md`
- **Verdict:** `resolved` (all exceptions handled) | `escalate` (unresolvable issues need ops manager)

### Phase 7: `generate-analytics` (agent: ops-analyst)
- Reads all data files for the day's operations
- Generates:
  - **Driver scorecards**: stops completed, on-time %, failed delivery rate, avg dwell time
  - **Route efficiency**: planned vs actual distance, planned vs actual duration, idle time
  - **Service metrics**: first-attempt delivery rate, customer satisfaction proxy (on-time + no damage)
  - **Operational summary**: total deliveries, success rate, exceptions by type, cost per delivery
- Writes `data/daily-analytics.json`
- Writes `documents/daily-dashboard.md`
- **Always advances** — final phase

### Phase 8: `abort-operations` (command phase)
- Triggered on unrecoverable failures
- Writes cancellation notice with reason

---

## Workflow Routing

```
validate-orders
  ├── advance → cluster-deliveries
  ├── rework → validate-orders (max 2)
  └── fail → abort-operations

cluster-deliveries → optimize-routes

optimize-routes
  ├── optimal → assign-drivers
  ├── acceptable → assign-drivers
  └── reroute → cluster-deliveries (max 3)

assign-drivers → track-progress

track-progress
  ├── completed → generate-analytics
  ├── exceptions → handle-exceptions
  └── rework → track-progress (max 5, simulate progress)

handle-exceptions
  ├── resolved → track-progress (resume tracking)
  └── escalate → generate-analytics (still generate end-of-day report)

generate-analytics (terminal — workflow complete)
abort-operations (terminal — workflow cancelled)
```

---

## Schedules

| Schedule ID | Cron | Purpose |
|---|---|---|
| `morning-route-planning` | `0 5 * * 1-6` | 5 AM daily (Mon-Sat) — process day's delivery batch |
| `midday-reoptimization` | `0 12 * * 1-6` | Noon — re-optimize remaining routes based on morning progress |
| `evening-analytics` | `0 19 * * 1-6` | 7 PM — generate daily performance analytics |

---

## MCP Servers

| Server | Package | Purpose |
|---|---|---|
| `filesystem` | `@modelcontextprotocol/server-filesystem` | Read/write all data and document files |
| `sequential-thinking` | `@modelcontextprotocol/server-sequential-thinking` | Structured reasoning for clustering and optimization |

---

## Seed Data Files

### `data/delivery-orders.json`
Batch of 35 delivery orders for a single day in Austin, TX metro area. Mix of:
- Residential (apartments, houses) — 70%
- Commercial (offices, stores) — 20%
- Bulk/heavy (furniture, appliances) — 10%

Each order: order_id, customer name, address, zip, lat/lng, package (weight, dims, type), time_window (morning/afternoon/anytime), special_instructions, priority (standard/express)

### `data/driver-fleet.json`
Fleet of 5 drivers with:
- Driver ID, name, phone
- Vehicle: type (van/truck), max_weight, max_volume, fuel_type
- Start location (depot or home address with lat/lng)
- Shift: start_time, end_time, max_hours
- Zone familiarity: list of zip codes they know well
- Performance history: on_time_pct, avg_deliveries_per_hour

### `data/service-area.json`
Austin metro service area:
- Depot location (central warehouse)
- Service zones (North, South, East, West, Downtown) with zip code mappings
- Average drive speeds by zone (downtown slower, suburban faster)
- Known problem areas (construction zones, gated communities, limited-access roads)

---

## README Outline

1. **What This Does** — one-paragraph overview
2. **How It Works** — pipeline diagram showing phase flow with decision branches
3. **Quick Start** — `ao daemon start`, enqueue a delivery batch
4. **Data Flow** — table of all JSON files and which phase reads/writes them
5. **Decision Points** — explain the three key verdicts and routing logic
6. **Customizing** — how to add drivers, change service area, adjust clustering params
7. **Schedules** — explain the three daily runs
8. **Domain Terms** — last-mile glossary (dwell time, first-attempt rate, etc.)

---

## Key Differentiators from Other Examples

- **Geographic clustering** — demonstrates intelligent data partitioning before optimization
- **Multi-level decision routing** — route quality feeds back to clustering, delivery failures feed back to tracking
- **Three scheduled runs per day** — shows AO handling a real daily operations cadence
- **Command phase** for abort — lightweight fallback without a full agent
- **Six agents across two model tiers** — Haiku for validation/analytics, Sonnet for optimization/decisions
- **Real-world operational domain** — last-mile delivery is a $100B+ industry problem
