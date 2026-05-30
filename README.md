# PSA Analytics Catalog

GitHub-hosted catalog of installable reports, dashboards, and custom report types for [Revecast PSA](https://github.com/Revecast/PSACore).

Installed at runtime by the **PSA Admin Setup Wizard** (`Analytics` tab) — admins pick the dashboards they want, the wizard pulls the metadata from this repo and deploys it to their org via Metadata REST API.

> **Why not ship these in the wizard package?**
> Early 2026 we tried packaging CRTs/reports inside the PSA Admin Setup Wizard (PSACore PR #103, reverted in `3215a25d`). Custom report types collided with legacy ForecastCloud-era unmanaged report types in mixed-namespace sandboxes — package install failed with *"base object cannot be changed"*. Moving to a runtime catalog (PSA-959) lets each admin opt in to specific bundles and lets us add new bundles without cutting a new wizard package version.

## Catalog structure

```
psa-analytics-catalog/
├── catalog.json                     # Master manifest — read first
└── bundles/
    ├── revenue-forecast-v2/
    │   ├── bundle.json              # Bundle-specific metadata
    │   ├── reportTypes/
    │   ├── reports/
    │   └── dashboards/
    └── …
```

Each bundle is a self-contained SFDX-style metadata tree that can be deployed independently with `sf project deploy start -d bundles/<bundle-id>`.

## catalog.json

The master manifest. The wizard fetches this at runtime and renders its Analytics picker from `categories[]` + `bundles{}`.

Schema:

| Field | Purpose |
|---|---|
| `schemaVersion` | Wizard checks compatibility before parsing |
| `minWizardVersion` / `minPSACoreVersion` | Surfaces in the picker UI as install gates |
| `categories[]` | Top-level tabs/sections in the picker |
| `bundles{}` | Map of `bundleId` → bundle definition |

### Bundle fields

| Field | Purpose |
|---|---|
| `name`, `description`, `tags` | Surface text in the picker |
| `category` | Which top-level category this belongs to |
| `primaryDashboard` | The face card of the bundle |
| `components` | Lists of dashboards / reports / report types / folders the bundle ships. The Apex installer reads this and pulls only the listed files |
| `dependsOn` | Other bundle IDs this one needs. The picker auto-selects dependencies when this bundle is checked. Used for shared-report dashboards like Delivery Capacity Reports ← Long Term Schedulin'. |

## Available bundles (v1.0)

| Bundle | Category | Components | Notes |
|---|---|---|---|
| Revenue Forecast v2 | Revenue | 1 dash, 6 reports, 1 CRT | Pipeline forecast with filters |
| Active Project Revenue Overview | Revenue | 1 dash, 3 reports, 1 CRT | Project financials |
| Time Monitoring Dashboard - Week | Delivery | 1 dash, 11 reports, 2 CRTs | Weekly time review |
| Long Term Schedulin' | Delivery | 1 dash, 10 reports, 1 CRT | Capacity outlook by role |
| Delivery Capacity Reports | Delivery | 1 dash | Reuses Long Term Schedulin' reports |

## Manual deploy (for testing)

```bash
sf project deploy start \
  --source-dir bundles/revenue-forecast-v2 \
  --target-org <alias>
```

Folders must exist in the target org first — create via REST:

```bash
# Report folder
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$INSTANCE/services/data/v60.0/sobjects/Folder" \
  -d '{"Name":"Revenue Reports","DeveloperName":"Revenue_Reports","Type":"Report","AccessType":"Public"}'

# Dashboard folder
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$INSTANCE/services/data/v60.0/sobjects/Folder" \
  -d '{"Name":"Revenue Dashboards","DeveloperName":"Revenue_Dashboards","Type":"Dashboard","AccessType":"Public"}'
```

## Compatibility

All metadata in this catalog references the **`Revecast` namespace**. The target org must have PSACore (≥ `0.11.0-2`) installed.

Mixed-namespace orgs (ForecastCloud + Revecast) need extra care — the wizard does collision detection on CRT names before deploying. See `docs/collision-strategy.md` (TBD).

## Adding a new bundle

1. Create `bundles/<bundle-id>/` with SFDX layout
2. Add an entry to `catalog.json` → `bundles{}`
3. Add the bundle ID to the relevant `categories[].bundleIds`
4. Tag a release in this repo — the wizard caches catalog.json by ETag
