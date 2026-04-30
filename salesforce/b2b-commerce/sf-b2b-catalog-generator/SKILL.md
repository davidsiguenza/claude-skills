---
name: sf-b2b-catalog-generator
description: Generate custom Salesforce B2B Commerce product catalogs (CSV import) for demos, tailored to a specific company or industry. Use when the user asks to "create a B2B catalog", "generate a Commerce demo catalog", "make products for a B2B Commerce demo", or similar requests involving the `b2bCatalog/` folder.
---

# B2B Commerce Catalog Generator

Generate a Salesforce B2B Commerce-ready product catalog as a CSV, personalized for a customer or industry, starting from minimal inputs. Outputs a file that can be loaded via B2B Commerce Advanced Import.

> **Part of the `sf-b2b-demo-builder` skill package.** If the user asks for "a complete B2B demo" or "store + catalog", invoke `.cursor/skills/sf-b2b-demo-builder/SKILL.md` first — it routes between this skill and `sf-b2b-store-generator` and shares context (target org, entitlement policy, brand profile, image-host CSP whitelist) so this skill's preflight (Step 1, 2b, 2c) can be skipped or pre-filled. When invoked standalone, this skill behaves as documented below — running its full preflight from scratch.

## Workflow at a glance

```
1. Ask user (entitlements + company/industry + extra context)
2. Research the customer/industry (web + Wikimedia)
2b. Pick target org via Salesforce CLI (always ask)
2c. Deploy CSP Trusted Sites for image domains (required or images don't render)
3. Design catalog structure (4 categories, ~20 products, variations only where natural)
3b. If variations exist → propose + ask → deploy Picklist fields + AttributeSets + SetItems
4. Source product images (Wikimedia direct URLs; drop products that can't hit 2 images)
5. Write 100–300 word descriptions
6. Produce the CSV using Python + csv.DictWriter
7. Verify (column count, image URLs, variation linkage) and report back
```

## Workflow

### Step 1 — Gather inputs from the user

Ask **four questions** at the start (in a single message, so the user can answer them all at once):

1. **Entitlement(s)** to associate with the products. They can provide 1 to 3 entitlement names. Example answers: `Regular Buyers`, `VIP Policy`, `Ascendum Entitlement Policy`.

2. **Company or industry** for the demo. Either:
   - Company name **+ URL** (e.g., `Ascendum — https://ascendum.pt/`), OR
   - An industry / sector (e.g., "fashion retail", "industrial chemicals", "medical devices").

3. **Image hosting strategy** — critical to ask up-front so images are consistent and URLs are stable on day one. Offer the four options below, and if the user picks one that needs credentials/IDs (Cloudinary, Salesforce Files, GitHub raw), ask for them in the same turn. See [IMAGE_HOSTING.md](references/IMAGE_HOSTING.md) for full recipes:
   - **A. Cloudinary** (recommended when user already has a cloud account). Ask for: cloud name, API key, API secret, target folder name. URLs are `res.cloudinary.com/<cloud>/image/upload/.../<folder>/<sku>.jpg`. Requires adding `res.cloudinary.com` to Trusted URLs.
   - **B. Direct hotlink from source** (Wikimedia / vendor CDN — what older flows did). Fastest, zero setup, but URLs can break if the source site changes or blocks hotlinking; add every source domain to Trusted URLs.
   - **C. Salesforce Files** (`ContentVersion` + `ContentDistribution`). Most portable (assets travel with the org), more laborious to automate, and the URLs are longer.
   - **D. GitHub raw / other user-provided CDN**. User supplies a repo + path; images go through `raw.githubusercontent.com`.
   - Default if the user has no preference and the catalog is for an internal demo: **B (direct hotlink from Wikimedia/vendor)**.

4. **Extra context (optional)** — invite them to share any constraints. **Include concrete examples so they know what to tell you**:
   - **Language** of the catalog (default: Spanish; can be English, Portuguese, etc.)
   - **Currency** — ISO 4217 code (USD, EUR, GBP, ...). If this catalog is being generated for an existing storefront created by `sf-b2b-store-generator`, read the `currency` field from `branding-work/<slug>/brand.json` and use it without re-asking; the currency must match the `Pricebook2.CurrencyIsoCode` of the store's pricebooks or the pricing engine will silently render "Price Unavailable". Only ask the user when running standalone with no store context.
   - **Number of products** (default: 20 base products across 4 categories)
   - **Specific categories** to focus on (e.g., "only electric vehicles", "skip services")
   - **Product types** within the industry (e.g., "B2B wholesale, not consumer retail")
   - **Tone / style** of descriptions (technical, premium/luxury, approachable)
   - **Price range** to anchor to (e.g., "enterprise tier, mostly $50k+")
   - **Number of price tiers**: default is `sale`, `original`, `VIP` (3 tiers in USD). User can ask for `Special Pricing` as a 4th tier or reduce to 2.

### Step 2 — Research

- If a **company URL** was provided: fetch the homepage and any obvious products/catalog pages to understand brands sold, categories, and extract real product names/specs/images where available.
- If only an **industry** was provided: use general knowledge of that industry's typical products, plus WebFetch to dominant vendors' Wikipedia pages, Wikimedia Commons categories, or vendor sites for real model names and images.

Expect many vendor sites to be JavaScript-heavy — WebFetch often returns generic category pages. **Fall back to Wikipedia / Wikimedia Commons** for real model lists and verifiable images; most real-world B2B brands have well-documented Commons categories.

### Step 2b — Pick the target Salesforce org

The skill deploys metadata into a connected org in Step 3b, so we need to know **which** org. **Always** ask, even if there is a default — picking the wrong org (e.g. a customer prod instead of a demo scratch) is hard to undo.

See [SF_CLI_ORG.md](references/SF_CLI_ORG.md) for the full list-and-select flow. Short version:

```bash
sf org list --json
```

Parse the result, show the user **aliases + usernames + instance URLs + which is default** in a short table, and ask which to use. Accept "skip" as an answer — the user may want to generate only the CSV without touching any org (in which case Step 3b is skipped and variations are dropped).

### Step 2c — Deploy CSP Trusted Sites for image domains

Salesforce orgs block external images by default through Content Security Policy (CSP). Any image URL from a domain that isn't in the org's Trusted URLs list **will not render**, even though the CSV import itself succeeds.

**Three-layer image-visibility gotcha** — if images show in Salesforce internal UI but not on the storefront, diagnose in this order:

1. **Org-level CSP** (`CspTrustedSite`): required for all images from external domains.
2. **Site-level CSP** (Experience Builder → Settings → Security / Trusted URLs): LWR Enhanced sites use Strict CSP by default and ignore the org-level rule. Change Script Security Level to Relaxed CSP or add the image domain at the site level. Republish.
3. **CMS Workspace language** (most-overlooked): the importer creates the image `ManagedContentVariant` records in the **default language of the site's CMS Workspace**. If the site renders in a different language, the variants aren't found and the images don't display. Check Setup → Digital Experiences → the site's CMS Workspace → Settings → Default Language. See "CMS language gotcha on SDO orgs" below if you find `Swedish`/`sv` or any other unexpected default.

### CMS language gotcha on SDO orgs (as of 2026-04-23)

**Problem**: Salesforce SDO (demo) orgs ship with their CMS Workspaces defaulted to **Swedish (`sv`)**. Once the Workspace is created, `DefaultLanguage` is effectively read-only from the UI — you can **add** additional languages but cannot change the default. The Commerce CSV importer creates all `ManagedContentVariant` records using the Workspace default, so images land in `sv` and the storefront (usually `en_US`) can't find them → blank images even after CSP and Trusted URLs are correct.

**Confirmed symptoms**:
- Org-level `CspTrustedSite` deployed and active.
- Site-level Relaxed CSP or Trusted URLs with `img-src` configured + site republished.
- `ManagedContent` records exist in the right Workspace.
- `ManagedContentVariant.Language = 'sv'` while the site runs in `en_US`.
- Pre-existing (non-imported) variants with `Language = 'en_US'` render fine — confirming CSP is not the issue.

**Workaround (manual, ~5 min — NOT automated by the skill, VALIDATED 2026-04-23)**:

The `ManagedContent` Translation Export/Import flow materializes the content in a second language without redoing the import. Steps:

1. In the CMS Workspace UI, select the affected content (the images imported by the Commerce CSV importer).
2. **Manage → Translations → Export**.
3. In "Language to Export", add the target language (e.g. **English (United States)**) and click **Export**.
4. Download the generated file from **Export & Import Status**.
5. Back in the Workspace → **Manage → Translations → Import** → upload the file → run Import.
6. After the import, the target language should be available and the storefront should render the images.

**Why this isn't automated**: the Connect API routes for `/connect/cms` (spaces/contents/variants/translations) return `NOT_FOUND` in SDO LWR Enhanced orgs as of this writing. `ManagedContentVariant` is read-only via REST, and `ManagedContent` deletion via Data API returns `INSUFFICIENT_ACCESS`. Until a reliable API path is validated, tell the user to run the Translations flow manually and log the experience.

**Alternative workarounds** (pick whichever fits the demo):
- Set the site's render language to `sv` (Experience Builder → Settings → Language). The images render but the whole site is now Swedish-flavored — OK for internal demos.
- Delete all imported `ManagedContent` from the Workspace UI, change the Workspace default language if the UI allows, and reimport the CSV.
- Accept the images are invisible for this demo and focus on product data / variations.

**Two-layer CSP gotcha**: the Experience Cloud storefront has its **own** CSP configuration, separate from the org-wide `CspTrustedSite`. Deploying only the org-level trusted site makes images visible in Salesforce internal UI but **not** in the public storefront.
- **Org-level** (`CspTrustedSite` metadata): see below.
- **Site-level** (Experience Builder → Settings → Security/Trusted URLs): must be done per site. LWR Enhanced / modern Commerce sites use "Strict CSP" by default and ignore org-level CspTrustedSite for image rendering. The user must either change the site's Script Security Level to **Relaxed CSP** (inherits org-level) OR add the image domain as a site-level Trusted URL with `img-src` checked, then **publish the site**.
- The site-level step is currently best done from the UI — LWR Enhanced sites don't retrieve as `ExperienceBundle` metadata, and automating it via CLI is unreliable. Tell the user explicitly they need to do this if images don't appear on the storefront.

For every unique domain that the catalog's images come from, deploy a `CspTrustedSite` metadata. The set of domains depends on the **image hosting strategy chosen in Step 1**:

- **Cloudinary** → `https://res.cloudinary.com`
- **Direct hotlink** → typically `https://upload.wikimedia.org` + any vendor CDN used
- **Salesforce Files** → no external CSP needed (Files are served from the same org)
- **GitHub raw** → `https://raw.githubusercontent.com`

Use the same sfdx project created earlier in the flow (`b2bCatalog/<Customer>/metadata/`):

```xml
<!-- force-app/main/default/cspTrustedSites/<Name>.cspTrustedSite-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<CspTrustedSite xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>Wikimedia Commons — source of product images for B2B Commerce demo catalog.</description>
    <endpointUrl>https://upload.wikimedia.org</endpointUrl>
    <isActive>true</isActive>
    <context>All</context>
</CspTrustedSite>
```

Keep it minimal — do NOT include `isApplicableToFrameAncestors/FrameSrc/StyleSrc/FontSrc` etc. Those flags are obsolete in modern API versions and cause parse errors. `context=All` is what makes it apply to the storefront.

Add to the `package.xml` manifest and deploy together with any custom fields from Step 3b:

```xml
<types>
  <members>Wikimedia_Commons</members>
  <name>CspTrustedSite</name>
</types>
```

Name the file with underscores (valid API names only). Tell the user which domains were trusted.

### Step 3 — Design the catalog structure

- **4 top-level categories** in `Category 1` that cover the main verticals of the customer/industry.
- Add subcategories within `Category 1` using the path syntax `Top/Sub` (e.g. `Construction Equipment/Excavators`). See CSV_SCHEMA.md "Category columns" — `Category 1` and `Category 2` are PARALLEL taxonomies, each with its own `/`-separated hierarchy. A common pattern: `Category 1 = <product-type hierarchy>`, `Category 2 = Brands/<Manufacturer>`.
- Plan ~5 products per category → ~20 base products total.
- **Variations**: only add them when they arise naturally from the product line (e.g., transmission options for tractors, pack sizes for consumable goods). **Do not force variations.** Variations are **extra rows**, not counted against the 20-product target, and share a `Variation Parent (StockKeepingUnit)`.

#### Mandatory "All Products" category linkage

**Every** product in the CSV **must** also be assigned to the catalog's top-level `All Products` category in addition to its specific category — otherwise the storefront mega-menu has no entry that surfaces every product. The `All Products` category is created up-front by `sf-b2b-store-generator` Phase B (or, when this skill runs standalone against an existing catalog, by the agent before importing).

**Default implementation: add a `Category 3` column with the fixed value `All Products` for every row.** The Advanced Import treats `Category 3` as an additional parallel taxonomy (same semantics as `Category 1`/`Category 2`), so every imported product gets linked to `All Products` alongside its specific category in a single pass. Example:

```csv
Product Name,SKU,Category 1,Category 2,Category 3,...
Fresón Fresco 500g,SF-FRES-001,Frescos/Frutas,Brands/Sunnie Foods Frescos,All Products,...
```

Variation child rows keep `Category 1/2/3` blank (inherit from parent), same rule as the other category columns.

**Fallback (older Advanced Import dialects that don't accept `Category 3`)**: omit the column and insert `ProductCategoryProduct` rows after the import — one per imported product pointing at `<ALL_PRODUCTS_CAT_ID>`:

```text
sf data create record -s ProductCategoryProduct -v "ProductId=<PROD_ID> ProductCategoryId=<ALL_PRODUCTS_CAT_ID>" -o <alias>
# Repeat for every imported product (or wrap in an Apex script for batch insert).
```

The All Products category Id is persisted in `branding-work/<slug>/store-ids.json` by Phase B of the store generator. When this skill runs standalone (no `store-ids.json`), resolve it with: `sf data query -q "SELECT Id FROM ProductCategory WHERE CatalogId = '<CATALOG_ID>' AND Name = 'All Products'" -o <alias>`. If the query returns 0 rows, **create the category first** before continuing.

### Step 3b — Variation attributes: propose, ask, deploy

**Only run this step if the catalog design has variations.** If there are no natural variations, skip — no further org changes needed.

B2B Commerce variations require **all FIVE** pieces (dropping any one breaks the importer):

1. **Picklist custom fields on `ProductAttribute`** (e.g., `Transmission__c`, `Power__c`) with `<restricted>true</restricted>` picklist values. Deployed as metadata.
2. **Field-Level Security (FLS)** granting read+edit on those fields to the Commerce permission sets — `B2B_Commerce_Cart_Upload` (critical), `B2B_Commerce_Order_Builder`, `B2BCommerce_Community_Access`. Without this, the importer rejects the field with `'<Field>__c' isn't a valid field for Product Attribute` even though it exists.
3. **`ProductAttributeSet` records** — one per variation family (e.g., `Valtra_T_Variations`, `Valtra_G125_Variations`). Created via `sf data create record`.
4. **`ProductAttributeSetItem` records** — one per (set × field) combo. Links each set to its fields. Missing these causes `The selected attribute set is empty`. Critically: the `Field` column expects the **CustomField Id** (e.g., `00Ng8000004eSheEAE`), NOT the API name string.
5. **CSV rows** reference `DeveloperName` in `Variation AttributeSet` and `__c` API names in `Variation Attribute Name N`.

See [VARIATIONS_METADATA.md](references/VARIATIONS_METADATA.md) for full details, exact commands and gotchas.

**Propose to the user — in plain language**:
1. Which variations exist (e.g., "Valtra T Series — transmission and power").
2. What will be created: Picklist custom fields on `ProductAttribute`, one `ProductAttributeSet` per family, and one `ProductAttributeSetItem` per (family × field) linkage.
3. Which org it will touch (from Step 2b).
4. That these can be deleted from Setup later.

**Then ask**: yes / no / skip. Same three outcomes as before:
- **Yes** → deploy + create records
- **No / Skip** → strip all variation rows AND clear `Variation AttributeSet` on parent rows

**Idempotency**: before deploying, query Tooling API for existing CustomFields and `ProductAttributeSet` records; reuse or warn on duplicates.

**Stale product check before reimport**: if the catalog has variations AND the user is rerunning the import (not the first time), query `Product2` by parent SKUs. If they exist with `ProductClass = 'Simple'`, **delete them** — the importer doesn't promote Simple → VariationParent, and those rows will fail with `Choose a variation parent product`. See VARIATIONS_METADATA.md "Stale products".

**Propose to the user — in plain language, no Salesforce-jargon**:
1. Which variations exist in the catalog (e.g., "Valtra T Series — transmission: HiTech / Versu / Direct CVT; power: 155 HP / 235 HP / 271 HP").
2. What the skill will create **on their behalf**: Picklist fields on the `ProductAttribute` standard object, one per attribute, with the listed picklist values. Plus one `ProductAttributeSet` record per variation family.
3. Which org it will touch (the one picked in Step 2b).
4. That these can be deleted from Setup later if unwanted.

**Then ask**: "OK to proceed? yes / no / skip variations entirely". Three outcomes:

- **Yes** → run the deployment. See [VARIATIONS_METADATA.md](references/VARIATIONS_METADATA.md) for the exact `sf` commands and metadata shapes. After successful deploy, the CSV uses the `__c` API names of the fields just created.
- **No / Skip** → remove all variation rows from the catalog plan. The CSV will only contain the base products. The `Variation AttributeSet` / `Variation Attribute Name/Value` / `Variation Parent` columns stay empty. The base product prices and images remain intact.
- **If the user skipped the org in Step 2b** → also skip variations (no target org to deploy to).

**Never proceed without explicit confirmation.** Deploying fields is reversible but still modifies the org.

**Idempotency check**: before deploying, run `sf sobject describe --sobject ProductAttribute --target-org <alias>` and check if any of the proposed fields already exist. If they do, reuse them (don't redeploy) and warn the user. Many orgs come pre-seeded with fields like `Color__c`, `Size__c`, `Length__c`, `Warranty_Options__c` — check these FIRST and reuse them if the catalog's variations fit; only deploy new fields for attributes that don't already exist.

### Step 4 — Source product images

See [IMAGES.md](references/IMAGES.md) for sourcing (where images come from) and [IMAGE_HOSTING.md](references/IMAGE_HOSTING.md) for hosting (where URLs in the CSV point to).

**Golden rule**: every product ships in the final CSV with ≥ 2 verified working image URLs. **If a product cannot get 2 real images, drop the product** — the user prefers fewer products over products with placeholder imagery.

**Match-quality review gate (mandatory, 2026-04-27 lesson)**: after the first automated pass (search → download → upload), print a table of `SKU | Product Name | matched alt text | URL` for every product. Many first-pass matches are semantically wrong even when the HTTP check passes — e.g. searching "potito fruta pack 4" on a supermarket site returns a "preparado fruta con galleta" product. Scan the table and refine the per-SKU search query for any mismatches, then rerun only those SKUs (the download script should be resume-safe via a `image_mapping.csv` checkpoint file). Do NOT continue to the CSV-rewrite step until every row has a plausible match.

### Step 5 — Write descriptions

- Length: **100–300 words per product**, with an average of ~150–200.
- Content: include real specs where known (power, capacity, dimensions), typical applications, key features, who it's for. Goal is to make the site's search productive.
- Style: follow the user-requested tone; default is professional and somewhat technical.

### Step 6 — Produce the CSV

See [CSV_SCHEMA.md](references/CSV_SCHEMA.md) for the exact column order, quoting rules and worked examples.

**Output path**: `b2bCatalog/<Customer-or-Sector>/<Customer-or-Sector>.csv`
- Example: `b2bCatalog/Ascendum/Ascendum.csv`
- **Do NOT** use the name `advancedCommerceImport.csv` — the user wants the filename to carry the customer/sector name.
- Create the directory first if it doesn't exist.

**Implementation tip**: don't hand-craft the CSV — write a short Python script in `/tmp/` that builds the CSV using `csv.DictWriter` with `quoting=csv.QUOTE_MINIMAL`. Hand-crafted CSV rows almost always end up with column-count mismatches due to commas inside product descriptions.

### Step 7 — Verify and report

After writing the file, always run a verification pass:

1. **Column-count check**: read the CSV back and assert every row has the same number of columns as the header.
2. **Image URL check**: HEAD/GET every unique image URL; if any fail, either replace with a working alternative or drop the product.
3. **Variation-field check** (if Step 3b ran): re-run `sf sobject describe` on the target org and confirm the fields referenced in the CSV exist.
4. **All Products linkage check**. Pre-import: confirm every base/parent row in the CSV has `Category 3 = All Products` (variation child rows are blank by design and inherit from the parent). Post-import (after the user completes the Advanced Import in the UI): query `ProductCategoryProduct` for every imported SKU and assert each one is linked to **both** its specific category **and** the `All Products` top-level category. If the Advanced Import dialect silently dropped `Category 3`, fall back to inserting the missing `ProductCategoryProduct` rows. Without this, the All Products mega-menu entry surfaces a partial subset of the catalog.

```text
sf data query -q "SELECT ProductId, ProductCategory.Name FROM ProductCategoryProduct WHERE ProductId IN (SELECT Id FROM Product2 WHERE StockKeepingUnit IN ('SKU-1','SKU-2',...))" -o <alias>
# Expect: every ProductId appears at least twice — once for its specific category, once for 'All Products'.
```

5. **Summary report** to the user:
   - Path of the generated file
   - Count of base products vs. variations
   - Products per category
   - Any products dropped (and why)
   - Target org (alias + instance URL) and a list of what was deployed (fields + records), or "no org changes" if skipped
   - Any caveats about image sourcing or price assumptions

Ask the user to review and confirm before considering the task done.

## Defaults reference

| Parameter | Default | Override if user specifies |
|-----------|---------|----------------------------|
| Language | Spanish | Any (English common for Iberian multi-national clients) |
| Currency | USD | Respect user choice; update header columns AND amounts |
| Products count | 20 base | User can request fewer/more |
| Categories | 4 top-level | Can vary if industry is narrower/broader |
| Variations | Only where natural | Never forced |
| Images per product | Min 2, max 5 | — |
| Price tiers | `sale`, `original`, `VIP` | 4th tier `Special Pricing` on request |
| Entitlements | 1 mandatory | Up to 3 |

## What NOT to do

- Never output the file as `advancedCommerceImport.csv` — always name it after the customer/sector.
- Never include a product whose only image is a generic placeholder or the family logo — drop it instead.
- Never link Wikimedia images via `/thumb/NNNpx-...` URLs — they are blocked for hotlinking (HTTP 403/429). Use the direct file URL (`/wikipedia/commons/X/XX/filename.ext`). See IMAGES.md.
- Never force variations to pad the product count.
- Never hand-format CSV rows by concatenating strings — always use Python's `csv` module.
- **Never auto-select an org** even if there's a default — always ask.
- **Never deploy metadata without explicit confirmation** — show the plan first, then ask.
- **Never trust an HTTP 200 on an image URL as proof of a correct match** — a supermarket search query can return the wrong product with a perfectly valid image. Always print a review table of `SKU | alt text | URL` after the first automated pass and refine mismatches before writing the CSV. See Step 4.
- **Never skip Step 1's image-hosting question** — changing the hosting strategy after upload means regenerating every CSV URL and reimporting. Ask up-front.
- Never use plain-text attribute names like `Transmission` in `Variation Attribute Name N` columns — they must be API names (`Transmission__c`) of fields that exist in the target org.
- **Never put variation attribute fields on `Product2`** — they go on `ProductAttribute`. This is the B2B Commerce standard data model.
- **Never populate `Variation AttributeSet` on a parent row without corresponding variation child rows** — it creates orphan references. If variations are dropped, clear this column too.
