# Last-Mile Delivery Optimizer — Agent Context

This is an AO workflow example demonstrating last-mile delivery operations automation.

## What This Repo Does

This pipeline takes a daily batch of delivery orders and runs them through a complete operations cycle:
1. **Validate** — normalize addresses, check package specs, flag issues
2. **Cluster** — group stops by geography into driver-sized routes
3. **Optimize** — sequence stops within each cluster for minimum drive time
4. **Assign** — match routes to available drivers by capacity and familiarity
5. **Track** — simulate delivery execution, flag exceptions
6. **Resolve** — disposition failed deliveries (reattempt/redirect/reschedule/return)
7. **Analyze** — driver scorecards, route efficiency, daily operations dashboard

## Directory Structure

```
.ao/workflows/      AO workflow definitions (agents, phases, schedules)
data/               JSON files — input seed data and pipeline outputs
documents/          Markdown outputs — route manifests, exception log, dashboard
```

## Data Files

### Input (seed — do not overwrite)
- `data/delivery-orders.json` — 35 delivery orders for Austin TX metro
- `data/driver-fleet.json` — 5 drivers with vehicles, shifts, zone familiarity
- `data/service-area.json` — zones, depot, lockers, known problem areas

### Pipeline Outputs (written by agents)
- `data/validated-orders.json` — orders with validation_status and normalized addresses
- `data/delivery-clusters.json` — geographic clusters with stop lists
- `data/optimized-routes.json` — sequenced stops with timing estimates
- `data/driver-assignments.json` — route-to-driver mappings
- `data/delivery-progress.json` — live delivery execution state
- `data/failed-deliveries.json` — exception resolutions
- `data/daily-analytics.json` — end-of-day performance metrics

### Document Outputs
- `documents/route-manifests.md` — printable driver sheets
- `documents/exception-log.md` — failed delivery log with dispositions
- `documents/daily-dashboard.md` — ops manager performance summary

## Key Rules for Agents

1. **Never overwrite seed data** — `delivery-orders.json`, `driver-fleet.json`, `service-area.json` are inputs
2. **Always read previous phase outputs** before writing — chain the data correctly
3. **Assign lat/lng from zip codes** using the zone data in service-area.json, not external APIs
4. **Respect package limits** — van max 150 lbs / 245 cu ft; truck max 400 lbs / 750 cu ft
5. **Driver shift is 7:30 AM - 4:30 PM** (9 hrs max) — routes must fit within this window
6. **Exception-handler must produce a disposition for every failed stop** — no unresolved exceptions

## Verdict Reference

| Phase | Verdict | Meaning |
|---|---|---|
| validate-orders | advance | Batch valid, proceed |
| validate-orders | rework | Too many issues, retry validation |
| validate-orders | fail | Batch unprocessable, abort |
| optimize-routes | optimal | All routes under 9 hrs, efficiency ≥75 |
| optimize-routes | acceptable | Routes feasible with minor issues |
| optimize-routes | reroute | Routes infeasible, redo clustering |
| track-progress | completed | All stops resolved |
| track-progress | exceptions | Failed stops need handling |
| track-progress | rework | Routes still active, advance simulation |
| handle-exceptions | resolved | All exceptions have dispositions |
| handle-exceptions | escalate | Ops manager escalation needed |

## Running This Example

```bash
# Full daily operations
ao queue enqueue --title "last-mile-optimizer" --workflow-ref delivery-operations

# Planning only (no execution simulation)
ao queue enqueue --title "last-mile-optimizer" --workflow-ref route-planning-only

# Analytics only (assumes delivery-progress.json exists)
ao queue enqueue --title "last-mile-optimizer" --workflow-ref analytics-only
```

## Service Area Quick Reference

| Zone | Zip Codes | Avg Speed | Notes |
|---|---|---|---|
| North Austin | 78753, 78758, 78759, 78613 | 25 mph | Tech parks, apartments |
| Central North | 78751, 78752, 78757 | 20 mph | Dense residential |
| Downtown/Core | 78701, 78702, 78722 | 15 mph | High traffic, loading docks |
| East Austin | 78702, 78721, 78741 | 18 mph | Mixed residential/new construction |
| South Austin | 78704, 78745, 78741 | 22 mph | Residential + S Congress commercial |
| West/Westlake | 78746, 78749, 78731 | 28 mph | Gated communities, hilly |
