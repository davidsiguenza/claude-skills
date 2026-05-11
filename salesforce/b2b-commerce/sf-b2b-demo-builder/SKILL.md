---
name: sf-b2b-demo-builder
description: >-
  Top-level router for Salesforce B2B Commerce demos. Use whenever the user says things like "create a B2B Commerce demo", "set up a complete B2B store with products", "I need a new commerce demo", "build me a B2B demo for client X", "use the existing SDO B2B store", "quick B2B demo", or "brand the existing B2B site". Routes to either full new-store build or the recommended seeded SDO branded demo.
---

# Salesforce B2B Commerce Demo Builder — Package Skill

This skill is the **entry point** for B2B Commerce demos. Keep it lightweight: choose the path, then hand off to the appropriate child skill.

## Children

| Child skill | Folder | Owns |
|---|---|---|
| **`sf-b2b-seeded-sdo-demo`** | `.cursor/skills/sf-b2b-seeded-sdo-demo/` | Recommended path. Reuses `SDO - B2B Commerce Enhanced`, applies customer branding, imports a customer catalog, cleans seeded catalog surface, configures buyer-group home variants, and validates Standard/Silver buyer experiences. |
| **`sf-b2b-store-generator`** | `.cursor/skills/sf-b2b-store-generator/` | Full build from zero. Creates a new Experience site + WebStore + catalog scaffold + buyer groups + pricebooks + checkout/tax/payment/shipping/login setup. Use only when a truly separate new store is required. |
| **`sf-b2b-catalog-generator`** | `.cursor/skills/sf-b2b-catalog-generator/` | Catalog generation and CSV/import metadata patterns. Used by both paths. |
| **`sf-b2b-shared`** | `.cursor/skills/sf-b2b-shared/` | Shared references, currently DigitalExperienceBundle branding patterns. |

> Both children also work standalone if invoked directly. This skill is the preferred entry point when the user's intent covers more than one of those scopes — or when they're not sure where to start.

## Phase 0 — The single routing question (mandatory, always run first)

**Hard rule for the agent:** before reading any child SKILL.md, ask the user the question below and **wait for the answer**.

> **What do you want to do for this B2B Commerce demo?**
> 1. **Full build from zero** — create a new Experience site + WebStore + catalog scaffold + buyer groups + pricebooks + branding, then optionally add products. **Known caveat:** this path currently may fail checkout validation because of shipping setup/alignment issues. Use only when a truly separate new store is required and budget time to debug shipping.
> 3. **Recommended: Hybrid branded seeded SDO demo** — reuse the existing SDO B2B store and seeded buyer/pricing/shipping/checkout setup, apply client visual branding to the existing site, then generate/import a client-specific catalog from the client site.

Echo the choice back ("OK, going with option 3 — hybrid branded seeded SDO demo for client X") so the user can correct before any work starts.

## Routing logic

### Path A — Full new storefront + optional catalog (option 1)

**Current status / caveat:** Path A is not the recommended default because checkout can fail on shipping setup/alignment in freshly created stores. Until the shipping workflow is revalidated end-to-end, prefer Path C for demos that need to be presentable quickly.

1. Read `.cursor/skills/sf-b2b-store-generator/SKILL.md` and follow it from **Phase 0 — Preflight questions** through **Phase H — Search index** and **Phase Z — Self-validation checklist**.
2. Do **not** run that skill's Phase I directly. If the user wants products after the store is ready, run `sf-b2b-catalog-generator` using the **Phase Bridge — context handoff** below, then run **Phase Bridge — post-import**.
3. Hand back the public URL, buyer credentials, validation plan, and whether the catalog was loaded.

### Path C — Hybrid seeded SDO branded catalog demo (option 3)

Read `.cursor/skills/sf-b2b-seeded-sdo-demo/SKILL.md` and follow it end-to-end. It owns seeded context resolution, visual branding, catalog import, seeded catalog cleanup, buyer-group home variants, and validation for this path.

## Phase Bridge — context handoff (Path A full-store catalog handoff)

Read these files from the workspace and use them to pre-fill the catalog skill's preflight answers:

| Source file | Field | Maps to in `sf-b2b-catalog-generator` |
|---|---|---|
| `branding-work/<slug>/store-ids.json` → `org.alias` | Salesforce org alias | Step 2b — *Pick the target Salesforce org* (do NOT re-ask, just confirm) |
| `branding-work/<slug>/store-ids.json` → `webStoreId` | WebStore Id | Used for post-import wiring + verifying images render on the right site |
| `branding-work/<slug>/store-ids.json` → `catalog.productCatalogId` | Catalog Id | Catalog rows reference this catalog (every `ProductCategory` belongs to it) |
| `branding-work/<slug>/store-ids.json` → `catalog.entitlementPolicyId` | Entitlement policy Id (resolve `Name` with one SOQL query) | Step 1 Q1 — *Entitlement(s)*. Pre-fill with the policy name; ask the user only if they want to add more entitlement policies |
| `branding-work/<slug>/store-ids.json` → `pricebooks.list / sale / vip` | List / Sale / VIP custom pricebook Ids | Used in CSV → `PricebookEntry` rows (one per SKU, four pricebooks total — Standard PB also needs an entry per SKU as the platform's source-of-truth requirement) |
| `branding-work/<slug>/brand.json` → `brandName`, `categoriesForCommerce`, `businessAreas`, `colors`, `languages`, etc. | Brand profile | Step 1 Q2 — *Company or industry*. Pre-fill with the brand name + URL; the agent already knows brands sold, categories, and image hosts |
| `branding-work/<slug>/brand.json` → `cspTrustedDomains` (or `branding.cspTrustedDomain` in store-ids.json) | Already-deployed image domain | Step 2c — skip the deploy if the domain is already trusted; only deploy NEW domains the catalog needs (typically `upload.wikimedia.org`) |
| `branding-work/<slug>/brand.json` → `imageHosting` (optional new field) | Hosting strategy chosen for the demo (`cloudinary` / `direct` / `salesforce-files` / `github-raw`) and any required IDs (cloud name, folder, repo, etc. — secrets NOT persisted) | Step 1 Q3 — *Image hosting strategy*. Pre-fill the mode; only ask for secrets the agent can't reuse from a keychain or env var. If absent from `brand.json`, ask the user from scratch. |
| `branding-work/<slug>/brand.json` → `currency` | ISO 4217 currency code chosen for the storefront (e.g. `EUR`, `USD`, `GBP`) — set in Phase 0 Q4 of `sf-b2b-store-generator` | Step 3 Q3 defaults — all `PricebookEntry` rows created by the catalog generator must use this currency; no re-ask needed, just confirm in the handoff summary. |

Echo the inferred values back to the user as a short confirmation block:

```text
I'll generate the catalog with these defaults (taken from the store I just created):
- Org alias:       cursorTesting
- Target WebStore: Ascendum Commerce Demo (0ZEg8000000N4XZGA0)
- Catalog:         Ascendum Commerce Demo Catalog (0ZSg80000005snlGAA)
- Entitlement:     Ascendum Commerce Demo Entitlement Policy (1Ceg80000002ABNCA2)
- Pricebooks:      List / Sale / VIP custom pricebooks (3 IDs from store-ids.json)
                   + org Standard PB (queried separately) for the platform requirement
- Brand source:    branding-work/ascendum-commerce-demo/brand.json (Ascendum, ascendum.pt)
- Image hosts already trusted: ascendum.pt
- Image hosting strategy (from brand.json, if present): Cloudinary → cloud `dbnwhls0i`, folder `ascendum`
- I will still ask: # of products, currency, language, tone, price range, whether to add Wikimedia, plus any hosting secrets not already in env/Keychain.

Confirm or override?
```

Wait for explicit confirmation. Then proceed into the catalog skill's Step 3 onwards.

## Phase Bridge — post-import (after a catalog CSV has been generated for any path)

The `sf-b2b-catalog-generator` skill owns CSV generation. When a target store is known, the orchestrator owns the import and final wiring against that store.

1. **Run the Advanced Import — automated path by default**.

   The B2B Commerce Advanced Import REST API (`POST /commerce/management/import/product/jobs`) is validated working in standard B2B Commerce Enhanced SDOs as of 2026-05-08. See `sf-b2b-catalog-generator/references/CSV_IMPORT_API.md` for the full recipe.

   1. **Probe** (Step 0): a single POST with `{}`. If it doesn't return `NOT_FOUND`, the endpoint is available. Cache `importApiAvailable: true|false` in `branding-work/<slug>/store-ids.json` so subsequent demos against the same org skip the probe.

   2. **If available**: upload `ContentVersion` using the base64 JSON variant by default → POST start → GET poll. Start uses `POST /commerce/management/import/product/jobs`; polling uses `GET /commerce/management/import/product/jobs/<jobId>`. Body shape is `{importConfiguration: {importSource, importSettings.category, importSettings.price.pricebookAliasToIdMapping, importSettings.media.cmsWorkspaceId}}` when the CSV has media columns — see CSV_IMPORT_API.md for the canonical wrapped JSON.

   3. **Validate the result**: terminal success can be `JobComplete` or `Completed`, and neither guarantees zero row errors. Inspect `errorMessages`, `warningMessages`, and counters (`productsCreated`, `productsUpdated`, `productAttributesCreated`, `pricebookEntriesCreated`, `productCategoryProductsCreated`, `productMediaCreated`). Treat any `errorMessages` as a failure to surface to the user. Warnings about blank optional media columns are usually benign; invalid URL warnings are not.

   4. **If not available** (probe returned `NOT_FOUND`) or **if the API call fails**: print the UI fallback:

   ```
   The automated import API is not available in this org (or returned an error).
   Please run the import manually:
   1. Open: <instance URL>
   2. Setup → Commerce → Stores → <Site Name> → Products → Import Products
   3. Upload: <absolute CSV path>
   4. Wait for green confirmation, then tell me when it's done.
   ```

2. **Wait for the user to confirm import success** (or detect `JobComplete` / `Completed` from the API), then run the post-import wiring that the storefront skill's Phase I and catalog skill's Step 7 document:
   - Verify `PricebookEntry` rows landed on every mapped pricebook for every SKU. Path A with a new store normally expects Standard + List + Sale + VIP. Path B/C with the seeded store normally expects Standard + Cirrus Price Book + Cirrus Silver Price Book.
   - Verify `ProductCategoryProduct` rows link every product to at least the catalog's "All Products" category.
   - Verify `CommerceEntitlementProduct` rows exist for the entitlement policy from Phase Bridge above. The validated Advanced Import creates these rows; do **not** insert duplicates unless verification proves the importer skipped them in this org.
   - Run `sf commerce search start --store-name '<Site Name>' --targetusername <alias>` to rebuild the search index, but respect the Commerce 5-minute cooldown between full rebuilds.
   - Run the pricing sync: `sf api request rest "/services/data/v66.0/connect/core-pricing/sync/syncData" --method GET --target-org <alias>`.
   - Run `sf community publish --name '<Site Name>' --target-org <alias>`.
3. **CMS workspace ↔ site channel link** — the catalog importer puts product images in the org's CMS workspace. For new stores, that workspace is **not** linked to the new site's channels by default (every Commerce site has its own `Community` + `PublicUnauthenticated` `ManagedContentChannel` rows). Without the link, products show without images on the storefront. Seeded SDO paths usually already have this link, but still verify it if `productMediaCreated > 0` and images do not render. The exact PATCH call is in `sf-b2b-store-generator` SKILL → *CMS workspace ↔ site channel: discover and link via CLI*. After linking, the user may need to publish content from the CMS workspace UI.
4. **CMS language gotcha (SDO orgs)** — see `sf-b2b-catalog-generator` SKILL → *CMS language gotcha on SDO orgs*. If the workspace defaults to `sv` (Swedish) and the site renders `en_US`, do the manual **Translations Export → Import** flow. There is no API workaround as of 2026-04.

## Cross-references

- This package skill is the **preferred** entry point for new B2B Commerce demos.
- Child skills also work standalone — invoke them directly if you only want one scope and you know which one.
- After this skill finishes Path A, `branding-work/<slug>/store-ids.json` is the source of truth for the new store IDs. Treat it as the per-demo "manifest"; both children read and write to it.
- After this skill finishes Path C, the seeded-store context manifest (if persisted) plus `b2bCatalog/<Customer>/<Customer>.csv` are the handover artifacts.

## Maintenance rule

When either child skill changes (new gotcha, new phase, renamed object, new field), update its own SKILL.md — **not** this one. The orchestrator only contains routing logic and the context-handoff schema; everything else lives in the children.
