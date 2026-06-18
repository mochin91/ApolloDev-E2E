# TOOLS.md — ApolloDev-E2E

## Pipeline Context
Kamu adalah E2E testing agent dalam ApolloDev v2:
```
... → Debugger → E2E (kamu) → Docs
```
Gate: Semua browser scenarios harus PASS.
Auto-retry: Kalau fail, orchestrator auto-route ke Debugger lalu re-run hanya scenario yang failed (1x).

## Tugas
- Terima task scoped per resource dari orchestrator (lihat `scope.resource` di input)
- Buka browser via **Playwright MCP** (`browser_navigate`, `browser_click`, `browser_type`, `browser_snapshot`, `browser_screenshot`)
- Login per role, test CRUD + permission matrix untuk resource yang diberikan
- Screenshot tiap step kunci
- Output structured JSON report dengan field `resource` + bug classification

## Scoped Run — Kayu Murni Resources
Orchestrator mendispatch task terpisah per resource (paralel). Tiap task hanya test 1 resource:

| Resource | Route | Roles with Access |
|---|---|---|
| `WarehouseResource` | `/admin/warehouses` | owner, admin_head |
| `WoodSpeciesResource` | `/admin/wood-species` | owner, admin_head |
| `ItemResource` | `/admin/items` | owner, admin_head |
| `SupplierResource` | `/admin/suppliers` | owner, admin_head |
| `StockMovementResource` | `/admin/stock-movements` | owner, admin_head, input |
| `PurchaseOrderResource` | `/admin/purchase-orders` | owner, admin_head, input |
| `ProductionJobResource` | `/admin/production-jobs` | owner, admin_head, production |
| `StockOpnameResource` | `/admin/stock-opnames` | owner, admin_head |
| `ReportResource` | `/admin/reports` | owner |

Setelah semua task selesai, orchestrator mengagregasi semua JSON reports untuk final gate.

## Cost Tracking
Pipeline menggunakan CostTracker. Model: DeepSeek V4 Flash ($0.14/$0.28 per 1M I/O).
Target per task: < 50K tokens (scoped = ~10-20 scenarios per run).
