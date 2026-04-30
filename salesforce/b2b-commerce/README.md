# Salesforce B2B Commerce Skills

Three skills to build complete B2B Commerce demos on Salesforce SDO / sandbox / DX orgs.

## Usage

Invoke `sf-b2b-demo-builder` and answer the routing question:
- **Store only** → routes to `sf-b2b-store-generator`
- **Catalog only** → routes to `sf-b2b-catalog-generator`
- **Both** → runs store first, then catalog with shared context (org alias, entitlement policy, brand profile)

## Skills

### `sf-b2b-demo-builder`
Entry point and orchestrator. Shares context between child skills to avoid re-asking questions.

### `sf-b2b-store-generator`
- Phase 0 preflight (3 questions)
- Phases A–H: Experience Site, WebStore, pricebooks, buyer groups, permission sets, branding, search index
- Phase Z: self-validation checklist (PASS/FAIL before handover)

### `sf-b2b-catalog-generator`
- Step 1 preflight, Steps 2–7: research, org selection, variations, images, descriptions, CSV, verify
- Output: `b2bCatalog/<Customer>/<Customer>.csv` + SFDX metadata for attributes/variations

## Requirements

- Salesforce CLI (`sf`) authenticated to target org
- SDO, sandbox, or DX org cloned from SDO (not an empty scratch org)
- `ADOBE_STOCK_API_KEY` env var (optional — enables Adobe Stock image sourcing)
