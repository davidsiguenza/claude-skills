# CSV Import API — Automated Advanced Commerce Import

**Purpose**: trigger the same B2B Commerce Advanced Import that runs from the Setup UI (`Commerce → Stores → <store> → Products → Import Products`), but via REST so the orchestrator can complete the full demo build without asking the user to click through Setup.

**Status (2026-05-08)**: ✅ **Validated end-to-end** in standard B2B Commerce Enhanced SDOs (`DentaidNew`, v66.0). 31/31 rows imported successfully (Dentaid catalog: 7 VariationParents + 19 Variations + 5 Simple). The endpoint exists and works without special permission sets beyond the standard B2B Commerce Admin.

**Lesson learned 2026-05-08**: the original BFG Supply build guide had the **wrong path** — `/commerce/catalog/products/import` does not exist. The correct path, validated in two independent docs and tested live, is `/commerce/management/import/product/jobs`. The BFG variant was likely an earlier or invented endpoint that never shipped. Always trust validated probes over documentation-by-AI.

## ⚠️ Critical corrections from end-to-end run (2026-05-08)

Live validation against `DentaidNew` SDO surfaced several corrections to earlier versions of this doc — **read them before running the flow**:

1. **Polling is GET, NOT POST** — earlier versions said both start and poll use POST. Wrong. Start = POST. Poll = **GET**. POST on the jobId endpoint returns `METHOD_NOT_ALLOWED — Allowed are GET,HEAD,PATCH`.

2. **Terminal status can be `JobComplete`, not just `Completed`** — earlier doc said only Completed/Failed. The API returns `JobComplete` when the job finishes successfully. Treat both as terminal-success.

3. **CommerceEntitlementProduct rows ARE created by the importer** (1 per product). Earlier `demo-builder` Phase Bridge claimed the importer didn't create them and recommended a manual post-step. False — the importer creates them. Skip the manual post-step.

4. **FLS Path A + Path B as documented IS NOT SUFFICIENT.** Insert into `B2B_Commerce_Cart_Upload + Order_Builder + Community_Access` (3 permsets) gives 5-8 permsets coverage; the importer still fails with `'X__c' isn't a valid field for Product Attribute`. **Mandatory fix: mirror the EXACT permset coverage of a pre-seeded field like `Size__c`** (16 permsets in tested SDO). See VARIATIONS_METADATA.md "FLS coverage mirror" section.

5. **Multipart upload for ContentVersion FAILS** in standard SDOs — `entity_content` returns `INVALID_FIELD`, two binary parts return `Cannot include more than one binary part`. Use **Variant B (base64 in JSON body)** — Variant A (curl multipart) doesn't work in these orgs.

6. **The importer DOES create ProductMedia rows from `Media * Url N` columns** — but ONLY when two pre-conditions are met:
   - `importSettings.media.cmsWorkspaceId` is included in the body (the CMS workspace where ManagedContent will be created and ProductMedia will reference). Without this key, the importer parses the URLs but silently skips creating ProductMedia/ManagedContent (`productMediaCreated: 0`, `cmsWorkspaceId: null` in response).
   - URLs are well-formed — `https://shop.example.com//path//to//image.jpg` (double slashes, common in PrestaShop-style URLs) is rejected by Salesforce's URL validator with `'Invalid URL: ...'` in `warningMessages`. The product still imports, but `productMediaCreated` stays at 0 for that row. **Strip duplicate slashes from CSV image URLs before upload.**

   The supported `Media * Url N` columns the importer recognizes (validated 2026-05-08): `Media Standard Url 1..5`, `Media Listing Url 1`, `Media Tile Url N`, `Media Banner Url N`, `Media Attachment Url 1..3`. If a column is missing/blank for a row the importer adds a benign `Add a URL for Media STANDARD N` warning — these are informational only, not failures.

   **Body shape with media** (the corrected canonical shape):
   ```json
   {
     "importConfiguration": {
       "importSource": {"contentVersionId": "<CV_ID>"},
       "importSettings": {
         "category": {"productCatalogId": "<CATALOG_ID>"},
         "price":    {"pricebookAliasToIdMapping": {"sale": "<PB_ID>"}},
         "media":    {"cmsWorkspaceId": "<MANAGED_CONTENT_SPACE_ID>"}
       }
     }
   }
   ```
   Resolve `cmsWorkspaceId` with: `sf data query -q "SELECT Id FROM ManagedContentSpace WHERE Name LIKE '<Site Name>%'"` — the site's Managed Content Space is auto-created by Phase A.

7. **`Failed` is NOT terminal-bad — re-import works as upsert.** A first-run Failed with partial successes does NOT prevent a subsequent re-import from completing the missing rows. The 12 successes from run 1 persisted, run 2 (after FLS fix) added the 19 children that had failed and reported `productsUpdated: 12, productsCreated: 19`. Idempotency confirmed.

8. **Search index requires 5 min cooldown between rebuilds** — `B2B_SEARCH_ADMIN_ERROR — You updated the search index too soon. Wait at least 5 minutes between full index updates.` Plan the post-import rebuild accordingly.

9. **Header names are exact** — the importer rejects friendly aliases. `Description`, `Media Standard Alt Text 1`, and `Media Standard Alt Text 2` failed in a live SDO run. Use `Product Description`, `Media Standard AltText 1`, and `Media Standard AltText 2` exactly as defined in `CSV_SCHEMA.md`.

10. **Seeded-SDO demos need catalog surface cleanup after import** — importing a client catalog into `SDO - B2B Commerce Enhanced` does not remove the original Cirrus/Solar seeded products or categories. To make the site look like a client demo, delete only the old `ProductCategoryProduct` rows for non-demo SKUs, then delete empty seeded `ProductCategory` rows. Do not delete old `Product2`, `PricebookEntry`, or `CommerceEntitlementProduct` rows unless explicitly requested; removing category links is sufficient to hide seeded products from PLP/menu/search after reindex. The full operational flow lives in `sf-b2b-seeded-sdo-demo`.

## The 5-step flow

1. Upload the CSV as a `ContentVersion` blob (base64 JSON body is the validated SDO path; multipart is documented only as a fallback if it works in your org).
2. Resolve the required IDs: `webstoreId`, `productCatalogId`, and the pricebook IDs you want each `Price (alias) CCY` column to map to.
3. `POST /services/data/v66.0/commerce/management/import/product/jobs` with the wrapped `importConfiguration` body.
4. Poll the same endpoint with the returned job Id using GET every 10–15 seconds until `status` is one of `JobComplete`, `Completed`, or `Failed`.
5. Inspect counters and `errorMessages` to confirm zero row errors before continuing.

## Endpoints

| Step | Method | Path |
|---|---|---|
| Upload CSV | `POST` (JSON body w/ base64) | `/services/data/v66.0/sobjects/ContentVersion` |
| Start import | `POST` | `/services/data/v66.0/commerce/management/import/product/jobs` |
| Poll status | **`GET`** | `/services/data/v66.0/commerce/management/import/product/jobs/<jobId>` |
| Cancel | `PATCH` | `/services/data/v66.0/commerce/management/import/product/jobs/<jobId>` with `{"status":"Canceled"}` |

**Method gotcha**: the POST endpoint without jobId starts a new job. The endpoint **with jobId** accepts `GET / HEAD / PATCH` only — POST returns `METHOD_NOT_ALLOWED — Allowed are GET,HEAD,PATCH`. Use GET for polling.

## Step 0 — Lightweight probe (optional but recommended)

A single POST with `{}` to confirm the endpoint exists in the target org. If your skill is running against an org type for the first time, run this once and cache the result in `branding-work/<slug>/store-ids.json` as `importApiAvailable`.

```bash
PROBE=$(sf api request rest --method POST \
  --header "Content-Type: application/json" \
  --body '{}' \
  "/services/data/v66.0/commerce/management/import/product/jobs" \
  --target-org <alias> 2>&1 | grep -oE '"errorCode":\s*"[^"]+"|"status":\s*"[^"]+"' | head -1)

case "$PROBE" in
  *NOT_FOUND*)
    USE_API=false
    echo "❌ Import API not available in this org. Fall back to UI."
    ;;
  *)
    USE_API=true
    echo "✅ Import API responded — proceeding."
    ;;
esac

# Cache the result
if [[ -f "branding-work/<slug>/store-ids.json" ]]; then
  jq --argjson available "$USE_API" '.importApiAvailable = $available' \
    "branding-work/<slug>/store-ids.json" > /tmp/store-ids.json && \
    mv /tmp/store-ids.json "branding-work/<slug>/store-ids.json"
fi
```

**What "endpoint exists" looks like**: an empty-body POST returns a job record with `"status": "Failed"` — that means the endpoint accepted the call, parsed the body, and ran the job to failure (no `contentVersionId`). This is success for the probe.

**What "endpoint missing" looks like**: `{"errorCode": "NOT_FOUND", "message": "The requested resource does not exist"}`.

## Step 1 — Upload the CSV as ContentVersion

The Advanced Import expects the CSV as a `ContentVersion` (Salesforce Files) blob. Two upload variants — pick the one that fits your environment.

### Variant A — `curl` multipart (recommended)

```bash
CSV_PATH="b2bCatalog/<Customer>/<Customer>.csv"
CUSTOMER="<Customer>"

# Resolve auth + user
eval "$(sf org display --target-org <alias> --json | jq -r '.result | "INSTANCE_URL=\(.instanceUrl) ACCESS_TOKEN=\(.accessToken)"')"
USER_ID=$(sf org display user --target-org <alias> --json | jq -r '.result.id')

# Multipart POST: 'entity_document' is JSON metadata, 'entity_content' is the file
RESP=$(curl -sS -X POST "$INSTANCE_URL/services/data/v66.0/sobjects/ContentVersion" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "entity_content=@${CSV_PATH};type=text/csv" \
  -F "entity_document={\"Title\":\"${CUSTOMER} Catalog Import\",\"PathOnClient\":\"$(basename "$CSV_PATH")\",\"FirstPublishLocationId\":\"${USER_ID}\"};type=application/json")

CONTENT_VERSION_ID=$(echo "$RESP" | jq -r '.id')
echo "ContentVersion: $CONTENT_VERSION_ID"
```

### Variant B — base64 JSON upload (when curl multipart isn't available)

```bash
CSV_BASE64=$(base64 -i "$CSV_PATH" | tr -d '\n')
RESP=$(sf api request rest --method POST \
  --header "Content-Type: application/json" \
  --body "{\"Title\":\"${CUSTOMER} Catalog Import\",\"PathOnClient\":\"$(basename "$CSV_PATH")\",\"VersionData\":\"${CSV_BASE64}\"}" \
  "/services/data/v66.0/sobjects/ContentVersion" \
  --target-org <alias>)

CONTENT_VERSION_ID=$(echo "$RESP" | jq -r '.id')
```

Variant B is slower for large CSVs (base64 inflates 33% and the entire body sits in memory) but works in environments without `curl -F`. The job endpoint accepts ContentVersion Ids from either variant.

## Step 2 — Resolve required IDs

The import job needs three or four IDs depending on how the CSV is structured.

```bash
# WebStore Id — usually one B2B store per SDO
WEBSTORE_ID=$(sf data query --target-org <alias> \
  --query "SELECT Id FROM WebStore WHERE Type = 'B2B' LIMIT 1" --json \
  | jq -r '.result.records[0].Id')

# Product Catalog Id — the catalog the products will land in
PRODUCT_CATALOG_ID=$(sf data query --target-org <alias> \
  --query "SELECT Id FROM ProductCatalog LIMIT 1" --json \
  | jq -r '.result.records[0].Id')

# Standard pricebook Id — required as the source-of-truth row
STANDARD_PB_ID=$(sf data query --target-org <alias> \
  --query "SELECT Id FROM Pricebook2 WHERE IsStandard = true LIMIT 1" --json \
  | jq -r '.result.records[0].Id')

# Buyer-group pricebooks the CSV's Price (alias) columns map to
SALE_PB_ID=$(sf data query --target-org <alias> \
  --query "SELECT Id FROM Pricebook2 WHERE Name = 'Cirrus Price Book'" --json \
  | jq -r '.result.records[0].Id')
VIP_PB_ID=$(sf data query --target-org <alias> \
  --query "SELECT Id FROM Pricebook2 WHERE Name = 'Cirrus Silver Price Book'" --json \
  | jq -r '.result.records[0].Id')
```

If your demo uses the storefront created by `sf-b2b-store-generator`, the IDs are already in `branding-work/<slug>/store-ids.json` — read them from there instead of querying.

**Standard SDO buyer-group pricebooks** (pre-seeded in B2B Commerce Enhanced SDOs):
- `Cirrus Price Book` — for `Cirrus Buyer Group` (e.g., Lauren Bailey / Omega Inc.)
- `Cirrus Silver Price Book` — for `Cirrus Silver Buyer Group` (elevated tier)

If the catalog targets the standard SDO buyer persona (default behavior), reuse these — don't create new pricebooks. See `references/SDO_DEMO_PERSONA.md` for the full pre-seeded persona.

## Step 3 — Start the import job

The body shape requires a wrapper object. Three nested settings — `category`, `price`, `media` — control different parts of the import. **All three are needed for a complete import that includes ProductMedia rows from `Media * Url N` columns.** Skipping `media.cmsWorkspaceId` causes the importer to silently drop image URLs (productMediaCreated stays at 0).

```bash
# Resolve the CMS workspace tied to the site (auto-created by Phase A of sf-b2b-store-generator)
CMS_WORKSPACE_ID=$(sf data query --target-org <alias> --json \
  --query "SELECT Id FROM ManagedContentSpace WHERE Name LIKE '<Site Name>%'" \
  | jq -r '.result.records[0].Id')

JOB_RESP=$(sf api request rest --method POST \
  --header "Content-Type: application/json" \
  --body "{
    \"importConfiguration\": {
      \"importSource\": {
        \"contentVersionId\": \"${CONTENT_VERSION_ID}\"
      },
      \"importSettings\": {
        \"category\": {
          \"productCatalogId\": \"${PRODUCT_CATALOG_ID}\"
        },
        \"price\": {
          \"pricebookAliasToIdMapping\": {
            \"sale\": \"${SALE_PB_ID}\",
            \"original\": \"${STANDARD_PB_ID}\",
            \"VIP Pricing\": \"${VIP_PB_ID}\"
          }
        },
        \"media\": {
          \"cmsWorkspaceId\": \"${CMS_WORKSPACE_ID}\"
        }
      }
    }
  }" \
  "/services/data/v66.0/commerce/management/import/product/jobs" \
  --target-org <alias>)

JOB_ID=$(echo "$JOB_RESP" | jq -r '.jobId // .id')
echo "Started job: $JOB_ID"
```

**`webstoreId` is NOT a valid key** at any level of `importConfiguration` — sending it returns `JSON_PARSER_ERROR: Unrecognized field "webstoreId"`. The importer learns the catalog/pricebook/CMS context from the IDs you pass, never from a webstoreId.

### Strip duplicate-slash URLs before upload

Salesforce's URL validator rejects `https://shop.example.com//path//image.jpg` with `Invalid URL: ...` in `warningMessages` and skips ProductMedia creation for that row. PrestaShop-style sites commonly emit such URLs — clean them in the build script:

```python
import re
url = re.sub(r"(?<=https?://[^/])(/)+", "/", url)  # collapse repeated slashes after domain
url = re.sub(r"(?<!:)//+", "/", url)              # any other // outside the scheme
```

### Price alias mapping rule (critical)

The CSV's price columns use the format `Price (<alias>) <CURRENCY>`. The alias text inside the parentheses **must match** a key in `pricebookAliasToIdMapping`. Examples:

| CSV column | Alias key | Maps to |
|---|---|---|
| `Price (sale) USD` | `sale` | The buyer-group pricebook for "the price the buyer sees" |
| `Price (original) USD` | `original` | Standard Pricebook (or a "list" / strikethrough pricebook) |
| `Price (VIP Pricing) USD` | `VIP Pricing` | The VIP buyer-group pricebook |
| `Price (Special Pricing) EUR` | `Special Pricing` | A 4th-tier pricebook if used |

**Currency in the column header is informational** — the actual currency is taken from the pricebook's `CurrencyIsoCode`. The header just helps a human reading the CSV.

If you reference an alias in the CSV that isn't in `pricebookAliasToIdMapping`, the import fails for those rows with `Specify a valid price book alias`.

## Step 4 — Poll for completion

The job is async. Poll the **same endpoint** the start used, with the jobId appended. Start is POST; poll is **GET**.

```bash
for i in {1..30}; do
  RESP=$(sf api request rest --method GET \
    "/services/data/v66.0/commerce/management/import/product/jobs/${JOB_ID}" \
    --target-org <alias>)

  STATUS=$(echo "$RESP" | jq -r '.status')
  echo "[$(date +%H:%M:%S)] Job ${JOB_ID} — ${STATUS}"

  case "$STATUS" in
    JobComplete|Completed)
      echo "✅ Import succeeded:"
      echo "$RESP" | jq '{status, productsCreated, productsUpdated, productAttributesCreated, pricebookEntriesCreated, productCategoryProductsCreated, productMediaCreated, errorMessages, warningMessages}'
      break
      ;;
    Failed)
      echo "❌ Import failed:"
      echo "$RESP" | jq '{status, errorMessages, warningMessages}'
      exit 1
      ;;
    UploadComplete|InProgress|Pending)
      sleep 12
      ;;
    *)
      echo "Unexpected status: $STATUS"
      sleep 12
      ;;
  esac
done
```

### Status lifecycle

| Status | Meaning |
|---|---|
| `UploadComplete` | The CSV is parsed, the job is queued. |
| `InProgress` | A worker is creating Product2 / ProductAttribute / PricebookEntry / ProductCategoryProduct / ProductMedia rows. |
| `JobComplete` / `Completed` | Done. **Always check `errorMessages` and the per-counter outputs** — terminal success does not mean "every row succeeded". |
| `Failed` | Job-level failure (auth, body shape, missing required setting). Inspect `errorMessages`. |

### Counters returned

A successful response includes counters showing what the job did. Treat any unexpected zero as a red flag:

| Counter | What it counts |
|---|---|
| `productsCreated` / `productsUpdated` | Product2 rows |
| `productAttributesCreated` | ProductAttribute rows (one per variant per attribute) |
| `productAttributeSetProductsCreated` | Linkage records between attribute sets and parents |
| `pricebookEntriesCreated` | PricebookEntry rows across all configured pricebooks |
| `productCategoryProductsCreated` | Category linkage rows |
| `productMediaCreated` | ProductMedia rows from `Media * Url N` columns |
| `slugsCreated` | URL slug rows |
| `errorMessages` | Map of CSV row number → error string. Empty `{}` means clean import. |

## Step 5 — Inspect and reconcile

After the job's terminal status, run the smoke test from `SKILL.md` Step 7 #5 to confirm the counters match what the catalog plan expected. Specifically check:

- `productsCreated >= base_products + variations` from the CSV.
- `pricebookEntriesCreated >= productsCreated * len(pricebookAliasToIdMapping)`.
- `productCategoryProductsCreated >= base_products * 2` (one for the specific category, one for `All Products`).
- `errorMessages == {}` — if not, inspect each row error.

If any counter is short, **don't run the post-import wiring** (search index rebuild, channel link). Fix the root cause, re-import, and re-poll.

## Common errors

| Error / symptom | Cause | Fix |
|---|---|---|
| `NOT_FOUND` on the start call | Wrong path. Common mistake: using the BFG `commerce/catalog/products/import` instead of `commerce/management/import/product/jobs`. | Use the validated path. |
| `METHOD_NOT_ALLOWED` on the job status call | Wrong method for that endpoint. Start requires POST on `/jobs`; polling requires GET on `/jobs/<jobId>`. | Use the endpoint/method table above. |
| `"Specify a valid price book alias"` | A `Price (xxx) CCY` column in the CSV references an alias that's not in `pricebookAliasToIdMapping`. | Either drop the column from the CSV or add the alias to the mapping. |
| `"Provide a value for each variation attribute field"` | A multi-attribute set (e.g., `Size + Length`) has at least one variant where one of the fields is blank. The importer requires ALL fields populated for ALL variants of a multi-attr set. | Either populate every cell, or split the multi-attr set into single-attr sets (preferred — see ATTRIBUTE_SET_TEMPLATES.md note below). |
| `"Choose a variation parent product"` on parent rows + `"Enter a value in the Variation Parent field only when the value in the Product field isn't a variation parent"` on child rows | A pre-existing `Product2` with the parent SKU has `ProductClass = 'Simple'` from a prior partial import. The importer does NOT promote Simple → VariationParent. | Delete the stale Simple parent first. See VARIATIONS_METADATA.md "Stale products". |
| `'<Field>__c' isn't a valid field for Product Attribute` | The custom field exists but the importer's runtime user lacks FLS on it. | Add FLS to `B2B_Commerce_Cart_Upload`, `B2B_Commerce_Order_Builder`, `B2BCommerce_Community_Access`. See VARIATIONS_METADATA.md Step 3b. |
| `INSUFFICIENT_ACCESS_OR_READONLY` on the start call | Caller user doesn't have B2B Commerce admin permission set. | Assign `B2BCommerce_B2BCommerceAdminPsl` license + permission set, or run as System Administrator. |

## Single-attribute sets — strong preference

> ⚠️ The CSV import requires **every attribute in a set to have a value for every variant**. If a `ProductAttributeSet` has 2 fields (e.g., `Color__c` + `Size__c`) but a particular variant only varies on Color, the import fails with `"Provide a value for each variation attribute field"`.
>
> **Default to single-attribute sets** unless every variant truly varies on all axes. For a 12-variant grid `3 colors × 4 sizes`, you want ONE 2-attr set (every variant is a complete (color, size) pair). For a catalog with some color-only and some size-only families, use TWO single-attr sets.
>
> See ATTRIBUTE_SET_TEMPLATES.md for guidance on which templates are single-attr (most) and which need careful handling (Containers — size + color; Apparel — size + color).

## When to use this vs. the UI

| Use the API | Use the UI |
|---|---|
| Full demo flow where every step should be automated | The user prefers visual progress / cancellation control |
| Repeatable demos against the same org | First-time exploring an unfamiliar org |
| The probe in Step 0 returned the endpoint exists | The probe returned `NOT_FOUND` |

The API is **always preferable when it works** because it eliminates the manual "Setup → Commerce → Stores → Products → Import Products" click-through and the "wait for green banner" handoff. But never block the demo on it — the UI fallback is in `SKILL.md` Step 6b.

## Worked example — full sequence

```bash
#!/bin/bash
set -e
ALIAS="DentaidNew"
CSV="b2bCatalog/Dentaid/Dentaid.csv"
CUSTOMER="Dentaid"

# Resolve org context
eval "$(sf org display --target-org "$ALIAS" --json | jq -r '.result | "INSTANCE_URL=\(.instanceUrl) ACCESS_TOKEN=\(.accessToken)"')"
USER_ID=$(sf org display user --target-org "$ALIAS" --json | jq -r '.result.id')

WEBSTORE_ID=$(sf data query --target-org "$ALIAS" --query "SELECT Id FROM WebStore WHERE Type = 'B2B' LIMIT 1" --json | jq -r '.result.records[0].Id')
PRODUCT_CATALOG_ID=$(sf data query --target-org "$ALIAS" --query "SELECT Id FROM ProductCatalog LIMIT 1" --json | jq -r '.result.records[0].Id')
STANDARD_PB_ID=$(sf data query --target-org "$ALIAS" --query "SELECT Id FROM Pricebook2 WHERE IsStandard = true LIMIT 1" --json | jq -r '.result.records[0].Id')
SALE_PB_ID=$(sf data query --target-org "$ALIAS" --query "SELECT Id FROM Pricebook2 WHERE Name = 'Cirrus Price Book'" --json | jq -r '.result.records[0].Id')

# 1. Upload CSV
CV_RESP=$(curl -sS -X POST "$INSTANCE_URL/services/data/v66.0/sobjects/ContentVersion" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "entity_content=@${CSV};type=text/csv" \
  -F "entity_document={\"Title\":\"${CUSTOMER} Catalog Import\",\"PathOnClient\":\"$(basename "$CSV")\",\"FirstPublishLocationId\":\"${USER_ID}\"};type=application/json")
CV_ID=$(echo "$CV_RESP" | jq -r '.id')
[[ "$CV_ID" == "null" ]] && { echo "Upload failed: $CV_RESP"; exit 1; }
echo "Uploaded ContentVersion: $CV_ID"

# 2. Start import
JOB_RESP=$(sf api request rest --method POST --header "Content-Type: application/json" \
  --body "{
    \"importConfiguration\": {
      \"importSource\": {\"contentVersionId\": \"${CV_ID}\"},
      \"importSettings\": {
        \"category\": {\"productCatalogId\": \"${PRODUCT_CATALOG_ID}\"},
        \"price\": {\"pricebookAliasToIdMapping\": {
          \"sale\": \"${SALE_PB_ID}\",
          \"original\": \"${STANDARD_PB_ID}\"
        }}
      }
    }
  }" \
  "/services/data/v66.0/commerce/management/import/product/jobs" \
  --target-org "$ALIAS")
JOB_ID=$(echo "$JOB_RESP" | jq -r '.jobId // .id')
[[ "$JOB_ID" == "null" ]] && { echo "Job start failed: $JOB_RESP"; exit 1; }
echo "Started job: $JOB_ID"

# 3. Poll
for i in {1..30}; do
  RESP=$(sf api request rest --method POST --header "Content-Type: application/json" --body '{}' \
    "/services/data/v66.0/commerce/management/import/product/jobs/${JOB_ID}" \
    --target-org "$ALIAS")
  STATUS=$(echo "$RESP" | jq -r '.status')
  echo "[$(date +%H:%M:%S)] $STATUS"
  case "$STATUS" in
    Completed)
      echo "✅ Done"
      echo "$RESP" | jq '{productsCreated, productAttributesCreated, pricebookEntriesCreated, productCategoryProductsCreated, productMediaCreated, errorMessages}'
      exit 0
      ;;
    Failed)
      echo "❌ Failed:"
      echo "$RESP" | jq '{errorMessages, warningMessages}'
      exit 1
      ;;
    *)
      sleep 12
      ;;
  esac
done
echo "⏱️ Timeout. Check Setup → Commerce → Import history."
exit 1
```

Save as `b2bCatalog/<Customer>/run-import.sh`, `chmod +x`, run after the CSV is generated.
