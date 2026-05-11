---
name: sf-b2b-seeded-sdo-demo
description: >-
  Build a fast branded Salesforce B2B Commerce demo by reusing the existing SDO B2B Commerce Enhanced store instead of creating a new WebStore. Use when the user says "use the existing SDO B2B store", "hybrid branded seeded SDO demo", "quick B2B demo", "brand the existing B2B site", "reuse Cirrus buyer groups/pricebooks", or wants a customer-specific catalog and branding on the seeded SDO storefront.
---

# Salesforce B2B Commerce Seeded SDO Demo

Use this skill for the recommended fast demo path: keep the seeded SDO store's working buyer groups, pricebooks, checkout, tax, payment, shipping, and buyer users, then apply client branding and import a client-specific catalog.

This skill intentionally does **not** create a new WebStore. For full new-store provisioning, use `sf-b2b-store-generator`.

## Required references

Read these before making changes:

- `.cursor/skills/sf-b2b-shared/references/DIGITAL_EXPERIENCE_BRANDING.md`
- `.cursor/skills/sf-b2b-catalog-generator/SKILL.md`
- `.cursor/skills/sf-b2b-catalog-generator/references/CSV_IMPORT_API.md`
- `.cursor/skills/sf-b2b-catalog-generator/references/SDO_DEMO_PERSONA.md`
- `.cursor/skills/sf-b2b-catalog-generator/references/DEMO_SCENARIOS.md`

## Preflight

Ask only what is needed for the seeded-store flow:

1. **Target org**: list authenticated orgs and ask which alias to use.

   ```bash
   sf org list --json
   ```

2. **Client/company URL**: mandatory for both branding and catalog research.
3. **Catalog scope**: product count, language, price tone/range, and whether to use direct vendor images or another image hosting strategy.
4. **Confirm seeded-store path**: echo that no new WebStore, buyer groups, pricebooks, shipping, tax, payment, or buyer users will be created unless a validation check finds missing seeded prerequisites.

## Phase A — Resolve seeded SDO context

Resolve IDs live. Names are hints; trust IDs from the org.

```bash
sf data query -q "SELECT Id, Name, Type, DefaultLanguage, SupportedCurrencies FROM WebStore WHERE Name = 'SDO - B2B Commerce Enhanced' OR Type = 'B2B' ORDER BY Name" -o <alias>
sf data query -q "SELECT ProductCatalogId, ProductCatalog.Name, SalesStoreId FROM WebStoreCatalog WHERE SalesStoreId = '<WEBSTORE_ID>'" -o <alias>
sf data query -q "SELECT Id, Name FROM CommerceEntitlementPolicy WHERE Name = 'Cirrus Entitlement Policy'" -o <alias>
sf data query -q "SELECT Id, Name, IsStandard, CurrencyIsoCode FROM Pricebook2 WHERE IsStandard = true OR Name IN ('Cirrus Price Book','Cirrus Silver Price Book')" -o <alias>
sf data query -q "SELECT Id, Name FROM ManagedContentSpace WHERE Name LIKE '%Commerce%' OR Name LIKE '%SDO%' ORDER BY Name" -o <alias>
sf data query -q "SELECT Id, Name FROM Account WHERE Name IN ('Omega, Inc.','Acme Partners','Displaytech')" -o <alias>
sf data query -q "SELECT BuyerGroup.Name, BuyerGroupId, Buyer.Name FROM BuyerGroupMember WHERE Buyer.Name IN ('Omega, Inc.','Acme Partners','Displaytech') ORDER BY Buyer.Name, BuyerGroup.Name" -o <alias>
```

Expected seeded defaults:

- WebStore: `SDO - B2B Commerce Enhanced`
- Catalog: `B2B Commerce Catalog`
- Entitlement policy: `Cirrus Entitlement Policy`
- Pricebooks:
  - `original` -> org Standard Price Book
  - `sale` -> `Cirrus Price Book`
  - `VIP Pricing` -> `Cirrus Silver Price Book`
- Standard buyer group: `Cirrus Buyer Group`
- Silver buyer group: `Cirrus Silver Buyer Group`
- Standard-only validation account: usually `Displaytech`
- Silver-only validation account: usually `Acme Partners`
- Mixed account warning: `Omega, Inc.` can belong to both Cirrus groups, so it is not a clean Standard vs Silver visibility test account.

Persist a manifest when useful:

```text
branding-work/<client-slug>/store-ids.json
```

Minimum shape:

```json
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
  "cmsWorkspaceId": "<MANAGED_CONTENT_SPACE_ID>"
}
```

If a required seeded record is missing, stop and report the missing prerequisite. Do not create replacements silently.

## Phase B — Client research and brand profile

Fetch the client site and capture a reusable brand profile under:

```text
branding-work/<client-slug>/brand.json
```

Capture:

- Brand name and client URL.
- Primary and secondary colors.
- Logo source URL or generated StaticResource name.
- Hero/tagline/copy ideas.
- Product categories and product examples for catalog generation.
- Image domains that need `CspTrustedSite`.
- Sustainability/CSR phrasing for parallax/custom content.
- Catalog image strategy.

## Phase C — Brand the existing SDO site

Follow `.cursor/skills/sf-b2b-shared/references/DIGITAL_EXPERIENCE_BRANDING.md`.

Seeded SDO demos must leave no obvious Cirrus/SDO energy content on the home page:

- Replace logo via StaticResource and `/sfsites/c/resource/<ResourceName>`.
- Apply colors to every branding set.
- Replace hero, product/category banners, right panels, login/register logos, footer/header promo text, testimonial text, and browser title.
- Search and remove stock terms/images:

  ```text
  assets/images/
  sfdc-ckz-b2b.s3.amazonaws.com/SDO/Commerce_Images
  Emergency Prep
  Discover Nature
  Solar
  Energy
  Wind
  Battery
  Cirrus
  ```

- Replace seeded product tile leftovers such as `Emergency Prep`.
- Replace `c:parallaxcmp` stock title `Discover Nature's Energy` with brand CSR/sustainability copy.
- Add buyer-group-specific home content when useful. Use `User.Commerce.BuyerGroups` + `Contains` + buyer group name. Do not use `User.AccountId`, `User.ProfileId`, `User.UserType`, or custom User/Contact fields.

Deploy only the target bundle and publish:

```bash
sf project deploy start --target-org <alias> --source-dir force-app/main/default/digitalExperiences/site/<SITE_BUNDLE> --wait 20
sf community publish --name 'SDO - B2B Commerce Enhanced' --target-org <alias>
```

Verify the published site visually before moving on. At minimum: logo, colors, hero, product/category banners, parallax copy, login/register logo, and Silver/Standard buyer-group variant if configured.

## Phase D — Generate catalog

Run `sf-b2b-catalog-generator` using the seeded context:

- Entitlement: `Cirrus Entitlement Policy`.
- Pricebook aliases:
  - `original` -> Standard Price Book
  - `sale` -> `Cirrus Price Book`
  - `VIP Pricing` -> `Cirrus Silver Price Book`
- CMS workspace: resolved Commerce workspace.
- Currency: infer from seeded pricebooks and confirm only if ambiguous.
- Image hosting: prefer the chosen strategy from `brand.json`; direct vendor hotlinks are acceptable for internal demos when domains are added as CSP Trusted Sites.

Use exact CSV headers from `CSV_SCHEMA.md`. Header aliases such as `Description` or `Media Standard Alt Text 1` are rejected by Advanced Import.

## Phase E — Import catalog

Use the automated Advanced Import API by default. See `CSV_IMPORT_API.md`.

High-level flow:

1. Upload CSV as `ContentVersion` using base64 JSON body.
2. Start import:

   ```text
   POST /services/data/v66.0/commerce/management/import/product/jobs
   ```

3. Include `importSettings.media.cmsWorkspaceId` so `ProductMedia` and `ManagedContent` are created.
4. Poll with GET:

   ```text
   GET /services/data/v66.0/commerce/management/import/product/jobs/<jobId>
   ```

5. Treat `JobComplete` and `Completed` as terminal success, but still inspect counters and `errorMessages`.

If the API is unavailable or fails, give the Setup UI fallback and wait for the user to confirm import completion.

## Phase F — Clean seeded catalog surface

Importing a client catalog into `SDO - B2B Commerce Enhanced` does not remove the original Cirrus/Solar products/categories. Hide the seeded catalog surface by deleting category links only.

1. Query all catalog product links:

   ```bash
   sf data query -q "SELECT Id, Product.StockKeepingUnit FROM ProductCategoryProduct WHERE ProductCategory.CatalogId = '<CATALOG_ID>'" -o <alias> --json
   ```

2. Delete only `ProductCategoryProduct` rows whose SKU does not belong to the new demo catalog prefix.

Do **not** delete old `Product2`, `PricebookEntry`, or `CommerceEntitlementProduct` rows unless the user explicitly requests destructive cleanup. Removing category links is enough to hide old products from PLP/menu/search after reindex.

3. Verify only new catalog SKUs remain visible:

   ```bash
   sf data query -q "SELECT Id, Name, StockKeepingUnit FROM Product2 WHERE Id IN (SELECT ProductId FROM ProductCategoryProduct WHERE ProductCategory.CatalogId = '<CATALOG_ID>') ORDER BY StockKeepingUnit" -o <alias> --json
   ```

4. Delete empty seeded `ProductCategory` rows, preserving:

- `All Products`
- every category with remaining `ProductCategoryProduct`
- ancestors of used categories
- structural client roots created by the import

Delete children before parents.

## Phase G — Refresh storefront

Publish, rebuild search, and sync pricing.

```bash
sf community publish --name 'SDO - B2B Commerce Enhanced' --target-org <alias>
sf commerce search start --store-name 'SDO - B2B Commerce Enhanced' --targetusername <alias>
sf api request rest "/services/data/v66.0/connect/core-pricing/sync/syncData" --method GET --target-org <alias>
```

Respect the Commerce 5-minute search index cooldown. If search returns `You updated the search index too soon`, report that cleanup is complete and rerun after the cooldown.

## Phase H — Validation

Run focused validation before handoff:

- Guest can browse catalog.
- Standard buyer sees normal home content.
- Silver/VIP buyer sees buyer-group-specific home content, if configured.
- Only new products/categories appear in PLP/navigation.
- Product images render.
- Prices render for Standard and Silver/VIP buyers.
- Search returns new products after index completes.
- Reorder/buyer flows from `DEMO_SCENARIOS.md` work where applicable.

Always provide the user a step-by-step manual test plan with the site URL and suggested buyer usernames.

## Maintenance rule

If a seeded-SDO demo reveals a new operational gotcha, update this skill first. If the gotcha is a general DigitalExperienceBundle branding pattern, also update `sf-b2b-shared/references/DIGITAL_EXPERIENCE_BRANDING.md`.
