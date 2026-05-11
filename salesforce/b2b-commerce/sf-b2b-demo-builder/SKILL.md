---
name: sf-b2b-demo-builder
description: >-
  Top-level entry point ("package skill") for building Salesforce B2B Commerce demos on an SDO / sandbox / DX org. Routes between three modes: full new store from zero, strict catalog build on the seeded SDO B2B Commerce store, or a hybrid branded demo that reuses the seeded SDO store while applying client visual branding and a client-specific catalog. Use whenever the user says things like "create a B2B Commerce demo", "set up a complete B2B store with products", "I need a new commerce demo", "build me a B2B demo for client X", "use the existing SDO B2B store", "quick B2B demo", or "brand the existing B2B site".
---

# Salesforce B2B Commerce Demo Builder — Package Skill

This skill is the **entry point** for building a B2B Commerce demo end-to-end. It routes between the storefront and catalog child skills, and shares context between them so the user only answers each preflight question once.

## Children

| Child skill | Folder | Owns |
|---|---|---|
| **`sf-b2b-store-generator`** | `.cursor/skills/sf-b2b-store-generator/` | Experience Cloud site (LWR), WebStore, empty `ProductCatalog`, 3 custom pricebooks (List/Sale/VIP), entitlement policy, Standard + VIP buyer groups + accounts + contacts + users, shipping profiles, tax/checkout/payment alignment with the SDO reference store, guest browsing, branding (logo via StaticResource using `/sfsites/c/resource/<ResourceName>`, color tokens on all brandingSets, hero/banner images with CSP Trusted Site, hero copy, navigation), search index. **Stops at "store is publishable but empty".** |
| **`sf-b2b-catalog-generator`** | `.cursor/skills/sf-b2b-catalog-generator/` | Catalog plan tailored to a customer/industry, web research, image sourcing (Wikimedia + vendor domains via `CspTrustedSite`), variation attributes (`ProductAttribute` picklist fields + `ProductAttributeSet` + `ProductAttributeSetItem`), final CSV ready for B2B Commerce Advanced Import (`b2bCatalog/<Customer>/<Customer>.csv`) plus org metadata for variations. **Stops at "CSV + variation metadata ready"; the user runs the Advanced Import in the Setup UI.** |

> Both children also work standalone if invoked directly. This skill is the preferred entry point when the user's intent covers more than one of those scopes — or when they're not sure where to start.

## Phase 0 — The single routing question (mandatory, always run first)

**Hard rule for the agent:** before reading any child SKILL.md, ask the user the question below and **wait for the answer**.

> **What do you want to do for this B2B Commerce demo?**
> 1. **Full build from zero** — create a new Experience site + WebStore + catalog scaffold + buyer groups + pricebooks + branding, then optionally add products. **Known caveat:** this path currently may fail checkout validation because of shipping setup/alignment issues. Use only when a truly separate new store is required and budget time to debug shipping.
> 3. **Recommended: Hybrid branded seeded SDO demo** — reuse the existing SDO B2B store and seeded buyer/pricing/shipping/checkout setup, apply client visual branding to the existing site, then generate/import a client-specific catalog from the client site.

Echo the choice back ("OK, going with option 3 — hybrid branded seeded SDO demo for client X") so the user can correct before any work starts.

**Do not offer the old option 2 in the user-facing question.** The strict seeded SDO catalog-only path is retained below as a reference/deprecated internal route, but the recommended workflow for customer demos is option 3.

## Routing logic

### Path A — Full new storefront + optional catalog (option 1)

This is the original full-build mode.

**Current status / caveat:** Path A is not the recommended default because checkout can fail on shipping setup/alignment in freshly created stores. Until the shipping workflow is revalidated end-to-end, prefer Path C for demos that need to be presentable quickly.

1. Read `.cursor/skills/sf-b2b-store-generator/SKILL.md` and follow it from **Phase 0 — Preflight questions** through **Phase H — Search index** and **Phase Z — Self-validation checklist**.
2. Do **not** run that skill's Phase I directly. If the user wants products after the store is ready, run `sf-b2b-catalog-generator` using the **Phase Bridge — context handoff** below, then run **Phase Bridge — post-import**.
3. Hand back the public URL, buyer credentials, validation plan, and whether the catalog was loaded.

### Path B — Strict seeded SDO catalog demo (deprecated reference only)

This path follows the BFG Supply / B2B Commerce Demo Builder pattern: reuse the seeded SDO B2B store and create only catalog-facing demo data. It is kept for historical/reference purposes only. **Do not offer it as a normal user option.** For customer-facing demos, use Path C so the site is visually branded and the old seeded catalog is cleaned up.

If explicitly requested by a maintainer, this path means: **do not rebrand the home page. Do not create a new WebStore. Do not create buyer groups, pricebooks, shipping, tax, payment, or buyer users unless validation proves the seeded store is missing required data.**

1. Read `sf-b2b-catalog-generator/SKILL.md`, `sf-b2b-catalog-generator/references/SDO_DEMO_PERSONA.md`, and, when useful, the user's BFG/B2B Demo Builder reference docs.
2. Pick the target org via the catalog skill's Step 2b (`sf org list --json`) and explicit user confirmation.
3. Resolve the seeded SDO context with **Seeded SDO context resolution** below.
4. Confirm the resolved defaults with the user. Ask only for catalog-specific inputs: company/industry or product theme, product count, language, image hosting strategy, price range/tone, target currency if the seeded pricebooks do not reveal one, and whether to include Wikimedia/vendor domains.
5. Run `sf-b2b-catalog-generator` with the seeded context:
   - Entitlement default: `Cirrus Entitlement Policy`.
   - Pricebook aliases: `original` → org Standard Price Book, `sale` → `Cirrus Price Book`, `VIP Pricing` → `Cirrus Silver Price Book`.
   - Buyer validation persona: Lauren Bailey / Omega Inc.
   - CMS workspace: the resolved Commerce workspace, so the import job can create `ProductMedia`.
6. Run **Phase Bridge — post-import**.
7. Validate the demo using `DEMO_SCENARIOS.md`: variant selector, faceted filtering, buyer-specific pricing, reorder where applicable, images, and search.

### Path C — Hybrid seeded SDO branded catalog demo (option 3)

This is the "best of both worlds" path: reuse the seeded SDO B2B store, buyer groups, pricebooks, entitlement, payment/tax/shipping, and users, but apply client visual branding and load a client-specific catalog. It should feel like a customer demo without paying the setup cost of a new store.

1. Read `sf-b2b-catalog-generator/SKILL.md`, `sf-b2b-catalog-generator/references/SDO_DEMO_PERSONA.md`, and `sf-b2b-store-generator/SKILL.md` **Phase F2 — Branding**.
2. Pick the target org via `sf org list --json` and explicit user confirmation.
3. Ask for the client/company URL. Client research is mandatory in this path because it feeds both branding and catalog generation.
4. Resolve the seeded SDO context with **Seeded SDO context resolution** below.
5. Apply **Visual branding on the existing SDO store** below. This is mandatory for Path C; do not continue with a "branded" demo while the home page, logo, colors, or login screens still show Cirrus.
6. Run `sf-b2b-catalog-generator` with the same seeded context and the client research gathered for branding.
7. Run **Phase Bridge — post-import**.
8. Run **Seeded catalog cleanup after Path B/C import** below so the storefront catalog and navigation show only the new client catalog plus `All Products`, not the original Cirrus/Solar seeded categories.
9. Validate both the storefront visual branding and the catalog buyer flows.

## Seeded SDO context resolution (shared by Path B and Path C)

Resolve with live SOQL, using names as hints but always trusting IDs from the target org:

```bash
sf data query -q "SELECT Id, Name, Type, DefaultLanguage, SupportedCurrencies FROM WebStore WHERE Name = 'SDO - B2B Commerce Enhanced' OR Type = 'B2B' ORDER BY Name" -o <alias>
sf data query -q "SELECT ProductCatalogId, ProductCatalog.Name, SalesStoreId FROM WebStoreCatalog WHERE SalesStoreId = '<WEBSTORE_ID>'" -o <alias>
sf data query -q "SELECT Id, Name FROM CommerceEntitlementPolicy WHERE Name = 'Cirrus Entitlement Policy'" -o <alias>
sf data query -q "SELECT Id, Name, IsStandard, CurrencyIsoCode FROM Pricebook2 WHERE IsStandard = true OR Name IN ('Cirrus Price Book','Cirrus Silver Price Book')" -o <alias>
sf data query -q "SELECT Id, Name FROM ManagedContentSpace WHERE Name LIKE '%Commerce%' OR Name LIKE '%SDO%' ORDER BY Name" -o <alias>
sf data query -q "SELECT Id, Name FROM Account WHERE Name = 'Omega, Inc.'" -o <alias>
sf data query -q "SELECT BuyerGroup.Name, BuyerGroupId FROM BuyerGroupMember WHERE Buyer.Name = 'Omega, Inc.'" -o <alias>
```

Build an in-memory context manifest. If useful for repeatability, persist it as `branding-work/sdo-b2b-commerce-enhanced/store-ids.json`; otherwise keep it in session. Minimum shape:

```text
{
  "org": {"alias": "<alias>"},
  "siteName": "SDO - B2B Commerce Enhanced",
  "webStoreId": "<WEBSTORE_ID>",
  "catalog": {
    "productCatalogId": "<CATALOG_ID>",
    "entitlementPolicyId": "<CIRRUS_POLICY_ID>",
    "entitlementPolicyName": "Cirrus Entitlement Policy"
  },
  "pricebooks": {
    "standard": "<STANDARD_PB_ID>",
    "sale": "<CIRRUS_PRICE_BOOK_ID>",
    "vip": "<CIRRUS_SILVER_PRICE_BOOK_ID>"
  },
  "cmsWorkspaceId": "<MANAGED_CONTENT_SPACE_ID>",
  "seededPersona": {
    "account": "Omega, Inc.",
    "contact": "Lauren Bailey",
    "entitlement": "Cirrus Entitlement Policy"
  }
}
```

If any required seeded record is missing, stop and report the missing prerequisite. Do not silently create a replacement unless the user explicitly chooses to repair the seeded store.

## Visual branding on the existing SDO store (Path C only)

Reuse the implementation details from `sf-b2b-store-generator` **Phase F2 — Branding**, but apply these extra safety rules because Path C modifies an existing shared SDO site:

1. **Retrieve only the seeded store bundle**. Discover the exact `DigitalExperienceBundle` name for `SDO - B2B Commerce Enhanced`; do not deploy every bundle in the workspace.
2. **Create a local backup before editing** under `branding-work/sdo-b2b-commerce-enhanced/backups/<timestamp>/`.
3. **Deploy only targeted metadata**:
   - The specific `DigitalExperienceBundle` folder for the seeded store.
   - Any required brand `StaticResource` for the logo.
   - Any required `CspTrustedSite` for logo / hero / banner image domains.
4. **Apply the maximum safe visual branding**:
   - Logo through `StaticResource` and `SiteLogo` / `_SiteLogoUrl` using the LWR-safe published path `/sfsites/c/resource/<ResourceName>` and `url(/sfsites/c/resource/<ResourceName>)`. Do not use the shorter `/resource/<ResourceName>` path for Commerce LWR branding; it may return `200` but still not render in the header.
   - Color tokens on **every branding set present in the retrieved bundle**, not just the common four. Seeded SDO B2B stores may include `Home_Header` in addition to `B2B_Commerce`, `B2B_Footer`, `B2B_Home_Banner`, and `B2B_Right_Panel`.
   - `styles.css` DXP variables and button/link colors.
   - Home hero copy, button labels, browser title, login/register/forgot-password logo references, testimonial/product-card text, and all visible home text that mentions Cirrus, solar, energy, batteries, wind, charging, or other seeded demo concepts.
   - Home hero, category, and right-panel banner images via `imageInfo.url`, with meaningful alt text and working image URLs from the client/catalog image set. Never leave the stock SDO/Cirrus image URLs in a Path C demo.
   - Buyer-group-specific home content when the demo has Standard vs Silver/VIP buyer groups: duplicate the target component tree in `sfdc_cms__view/home/content.json` and use `contentOperations.operations` rules with `resource: "User.Commerce.BuyerGroups"`, `operator: "Contains"`, and `value: "<Buyer Group Name>"`. This is the validated CLI-deployable equivalent of Experience Builder's **User > Commerce > Buyer Groups contains ...** visibility rule.
   - Navigation labels only when they are clearly brand-facing text and do not break category navigation.
5. **Strip deploy blockers from retrieved bundles**:
   - Remove `geoBotsAllowed` from the site JSON before deploy.
   - If the retrieved SDO bundle contains generated approval/FSL routes or views that fail deploy because an object such as `FSL__Custom_Gantt_Action__c` is not installed, remove only those route/view folders from the local deploy payload. They are unrelated to the storefront catalog and can block otherwise valid branding deploys.
6. **Do not change operational store wiring**: no pricebook restructuring, no shipping/tax/payment changes, no new buyer users, no WebStore ownership changes, and no destructive cleanup of seeded records.
7. **Repair only missing seeded buyer access that blocks the standard persona**. If `Omega, Inc.` is not a member of `Cirrus Buyer Group`, insert that `BuyerGroupMember` after explicit user confirmation. This preserves the seeded persona while making standard Cirrus pricing work.
8. Publish after deploy and verify the home page visually in an incognito window before considering Path C branding complete. At minimum verify: logo, primary colors, hero copy, major home cards, and login/register logo no longer show Cirrus. Also probe `https://<site-domain>/<site-path>/sfsites/c/resource/<ResourceName>` and confirm it returns the expected image content type.

### Buyer group home variants (validated 2026-05-11)

To make Standard and Silver/VIP buyers see different home hero content without clicking through Experience Builder, edit the retrieved `DigitalExperienceBundle` directly:

1. In `sfdc_cms__view/home/content.json`, find the component to personalize (for example the first `dxp_content_layout:banner` under the home content region).
2. Deep-copy that component tree, generate fresh UUIDs for every `id`, and change its text/image/button attributes for the Silver/VIP experience.
3. Insert the personalized component next to the default component.
4. Add two `contentOperations.operations` entries:

   ```json
   {
     "targetId": "<SILVER_BANNER_COMPONENT_ID>",
     "isHiddenOnOperationSuccess": false,
     "isActive": true,
     "rule": {
       "name": "<UUID>",
       "description": "Show Silver banner for Cirrus Silver Buyer Group",
       "criteriaType": "AllCriteriaMatch",
       "expressionCriteria": [{
         "resource": "User.Commerce.BuyerGroups",
         "operator": "Contains",
         "value": "Cirrus Silver Buyer Group",
         "criterionNumber": 1
       }]
     }
   }
   ```

   ```json
   {
     "targetId": "<DEFAULT_BANNER_COMPONENT_ID>",
     "isHiddenOnOperationSuccess": true,
     "isActive": true,
     "rule": {
       "name": "<UUID>",
       "description": "Hide default banner for Cirrus Silver Buyer Group",
       "criteriaType": "AllCriteriaMatch",
       "expressionCriteria": [{
         "resource": "User.Commerce.BuyerGroups",
         "operator": "Contains",
         "value": "Cirrus Silver Buyer Group",
         "criterionNumber": 1
       }]
     }
   }
   ```

5. Deploy the site bundle and publish:

   ```bash
   sf project deploy start --target-org <alias> --source-dir force-app/main/default/digitalExperiences/site/<SITE_BUNDLE> --wait 20
   sf community publish --name '<Site Name>' --target-org <alias>
   ```

6. Validate with one buyer account that is **only** in the Silver/VIP group and one buyer account that is **only** in the normal group. Do not use accounts that belong to both groups for this validation; they will match the Silver/VIP rule.

**Do not use** `User.AccountId`, `User.ProfileId`, `User.UserType`, or custom `User`/`Contact` fields for this rule. Those expressions deploy-fail in B2B Commerce LWR with `Enter a valid expression`. The validated metadata resource is exactly `User.Commerce.BuyerGroups` with `Contains` and the buyer group name.

## Seeded catalog cleanup after Path B/C import

Path B and Path C reuse the seeded SDO B2B store, whose catalog normally contains Cirrus/Solar products and categories. After the client CSV import succeeds, clean the storefront catalog surface so the site shows the new demo only.

1. **Remove old products from the catalog by deleting only `ProductCategoryProduct` links for non-demo SKUs**:

   ```bash
   sf data query -q "SELECT Id, Product.StockKeepingUnit FROM ProductCategoryProduct WHERE ProductCategory.CatalogId = '<CATALOG_ID>'" -o <alias> --json
   ```

   Filter in code: keep rows whose SKU starts with the new catalog prefix (for example `DENT-`) and delete all other `ProductCategoryProduct` rows. Do **not** delete the old `Product2` records, `PricebookEntry` rows, or `CommerceEntitlementProduct` rows unless the user explicitly asks for destructive cleanup. Removing the category links is enough to hide seeded products from the storefront catalog.

2. **Verify only the new products remain catalog-visible**:

   ```bash
   sf data query -q "SELECT Id, Name, StockKeepingUnit FROM Product2 WHERE Id IN (SELECT ProductId FROM ProductCategoryProduct WHERE ProductCategory.CatalogId = '<CATALOG_ID>') ORDER BY StockKeepingUnit" -o <alias> --json
   ```

   Expected: only the new catalog prefix appears.

3. **Verify every new product is linked to `All Products`**:

   ```bash
   sf data query -q "SELECT Product.StockKeepingUnit, ProductCategory.Name FROM ProductCategoryProduct WHERE ProductCategory.CatalogId = '<CATALOG_ID>' AND ProductCategory.Name = 'All Products' ORDER BY Product.StockKeepingUnit" -o <alias> --json
   ```

   Expected: one `All Products` row per imported product or variation parent that should appear in PLP navigation.

4. **Remove empty seeded categories, but preserve the client category tree**:
   - Keep `All Products`.
   - Keep every category with a remaining `ProductCategoryProduct` row.
   - Keep all ancestors of those used categories.
   - Keep structural client roots such as `Brands`, `Toothpastes`, `Mouthwashes`, `Toothbrushes`, or whatever roots the CSV created.
   - Delete empty seeded categories such as `Energy Products`, `Energy Services`, `Featured Products`, `Smart Buildings`, `Solar Solutions`, `Charging`, `Charging Accessories`, `Wind Systems`, `Batteries`, and `Warranty` when they have no remaining products.
   - Delete child categories before parents.

5. **Refresh the storefront**:
   - Run `sf community publish --name '<Site Name>' --target-org <alias>`.
   - Run `sf commerce search start --store-name '<Site Name>' --targetusername <alias>`, respecting the 5-minute full-index cooldown. If the command returns `You updated the search index too soon`, report that the data cleanup is complete and rerun the rebuild after the cooldown.

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
- The two child skills also work standalone — invoke them directly if you only want one scope and you know which one.
- After this skill finishes Path A, `branding-work/<slug>/store-ids.json` is the source of truth for the new store IDs. Treat it as the per-demo "manifest"; both children read and write to it.
- After this skill finishes Path B or Path C, the seeded-store context manifest (if persisted) plus `b2bCatalog/<Customer>/<Customer>.csv` are the handover artifacts.

## Maintenance rule

When either child skill changes (new gotcha, new phase, renamed object, new field), update its own SKILL.md — **not** this one. The orchestrator only contains routing logic and the context-handoff schema; everything else lives in the children.
