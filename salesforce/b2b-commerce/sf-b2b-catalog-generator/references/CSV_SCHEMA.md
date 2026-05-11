# CSV Schema — Salesforce B2B Commerce Advanced Import

The catalog CSV uses a superset of Salesforce's Advanced Commerce Import format. The original import only has `Media Standard Url 1`; we've extended it with `Media Standard Url 2..5` so every product can carry 2–5 images.

## Column order (fixed — the import tool is position-sensitive for some fields)

| # | Column | Notes |
|---|--------|-------|
| 1 | `Product Name` | The short display name |
| 2 | `Product Description` | 100–300 words, detailed, technical |
| 3 | `SKU` | Unique, often prefixed with brand/family (e.g. `VCE-EC220E`) |
| 4 | `Product Family` | Typically the manufacturer brand (e.g. `Volvo CE`, `Ponsse`) |
| 5 | `Category 1` | Hierarchical category path, e.g. `Construction Equipment/Excavators`. See "Category columns" below. |
| 6 | `Category 2` | A **parallel** (not nested) hierarchy — typically brand, e.g. `Brands/Volvo CE`. |
| 7 | `Category 3` | **Mandatory** — fixed value `All Products` on every base/parent row. Links every product to the catalog-wide "All Products" category that feeds the mega-menu. Blank on variation child rows (inherit from parent). |
| 8 | `Price (sale) USD` | Current sale price (what the user sees by default). Update header if currency != USD. |
| 9 | `Price (original) USD` | PVP / "list" / strikethrough price |
| 10 | `Price (VIP Pricing) USD` | Price for VIP tier |
| 11 | `Entitlement 1` | Required. Comes from the user's first question. |
| 12 | `Entitlement 2` | Optional |
| 13 | `Entitlement 3` | Optional |
| 14 | `Product isActive` | `TRUE` for every product in the catalog |
| 15 | `Variation AttributeSet` | Set only on the parent of a variation family; blank otherwise |
| 16 | `Variation Attribute Name 1` | Only on variation child rows |
| 17 | `Variation Attribute Value 1` | Only on variation child rows |
| 18 | `Variation Attribute Name 2` | Only on variation child rows |
| 19 | `Variation Attribute Value 2` | Only on variation child rows |
| 20 | `Variation Parent (StockKeepingUnit)` | The parent SKU, only on variation child rows |
| 21 | `Media Standard Url 1` | **Primary image** → `productDetailImage` group (PDP). Required. |
| 22 | `Media Standard AltText 1` | Alt text for image 1 |
| 23 | `Media Standard Url 2` | Second PDP image (required — if unavailable, drop the product) |
| 24 | `Media Standard Url 3` | Optional |
| 25 | `Media Standard Url 4` | Optional |
| 26 | `Media Standard Url 5` | Optional (max) |
| 27 | `Media Listing Url 1` | **Product List Image** → `productListImage` group (PLP thumbnail). Populate this or the PLP shows no image. Usually same as `Media Standard Url 1`. Only on base/parent rows — children inherit. |
| 28 | `Media Listing AltText 1` | Alt text for the PLP image |
| 29 | `Media Attachment Url 1` | Reserved for PDF/ZIP attachments (spec sheets, brochures) — NOT for images |
| 30 | `Media Attachment Url 2` | Reserved for attachments |
| 31 | `Media Attachment Url 3` | Reserved for attachments |

## Header names are strict

The Advanced Import rejects near-miss headers. These names have been validated in SDO B2B Commerce Enhanced orgs:

- `Product Description` is valid; `Description` is rejected.
- `Media Standard AltText 1` is valid; `Media Standard Alt Text 1` is rejected.
- `Media Listing AltText 1` is valid; keep it paired with `Media Listing Url 1`.
- `Media Standard Url 1..5`, `Media Listing Url 1`, and `Media Attachment Url 1..3` use `Url`, not `URL`.

If the API returns `Header '<name>' is invalid`, rewrite the CSV to this schema exactly and re-upload a new `ContentVersion`. Re-import is idempotent for already-created rows.

## Media column → ElectronicMediaGroup mapping (validated 2026-04-23)

The importer inspects the column name prefix `Media <UsageType> Url N` and assigns the image to the matching `ElectronicMediaGroup.UsageType`. UsageTypes supported by B2B Commerce:

| Column prefix | UsageType | DeveloperName | Role in storefront |
|---|---|---|---|
| `Media Standard Url N` | `Standard` | `productDetailImage` | Product Detail Page gallery |
| `Media Listing Url N` | `Listing` | `productListImage` | Product List Page (PLP) thumbnail |
| `Media Tile Url N` | `Tile` | `tileImage` | Tile grids / cart / compact views |
| `Media Banner Url N` | `Banner` | `bannerImage` | Banners |
| `Media Attachment Url N` | `Attachment` | `attachment` | PDFs, ZIPs, brochures |

**Critical**: if you only populate `Media Standard Url`, the PLP will be blank for every product. Always populate `Media Listing Url 1` on base products.

**Variation children**: do NOT populate the Listing column on variation child rows. The PLP shows the variation parent, which already has its own Listing image. Listing on children is wasted effort at best, or overrides the parent at worst.

**Implementation hint for skill**: in the build script, auto-populate `Media Listing Url 1 = Media Standard Url 1` and `Media Listing AltText 1 = Media Standard AltText 1` for every row where `Variation Parent (StockKeepingUnit)` is empty (i.e. base products and variation parents).

## Category columns (critical — easy to misread)

`Category 1`, `Category 2` and `Category 3` are **NOT** nested (i.e. Category 2 is NOT a subcategory of Category 1). They are three **parallel taxonomies** assigned to the same product. Each column uses `/` to express its OWN hierarchy. `Category 3` is reserved for the catalog-wide `All Products` mega-menu entry (see "Convention" below).

**Wrong** (flat values treated as sibling top-level categories in the storefront):
```
Category 1 = "Trucks"
Category 2 = "Electric"
```
This ends up with `Trucks` AND `Electric` as two separate top-level categories — confirmed in demo 2026-04-23 where the Category Workspace listed `Trucks`, `Electric`, `Excavators`, `Construction`, `Long Haul` all as top-level siblings.

**Right** (hierarchies expressed with `/`):
```
Category 1 = "Trucks/Electric"           # product-type hierarchy
Category 2 = "Brands/Volvo Trucks"       # brand hierarchy, parallel to Category 1
```

The importer parses `/` as a path separator and creates the parent/child category records automatically. The Alpine sample uses exactly this: `Category 1 = "Energy/Energy Bars"`, `Category 2 = "Brands/BigfootBar"`.

**Convention for this skill**:
- `Category 1` = product-type hierarchy, `<TopCategory>/<Subcategory>`.
- `Category 2` = brand hierarchy, `Brands/<ManufacturerBrand>`. Usually derived from `Product Family`.
- `Category 3` = the fixed value `All Products` on every base/parent row. This is mandatory — it's what links every product to the catalog-wide mega-menu entry. See SKILL.md "Mandatory All Products category linkage".
- **Variation child rows**: leave `Category 1`, `Category 2` and `Category 3` blank — they inherit from the parent.

**Build script pattern**:

```python
cat1 = row.get("Category 1", "")
cat2_sub = row.get("Category 2", "")  # originally written as flat subcategory
if cat1 and cat2_sub:
    row["Category 1"] = f"{cat1}/{cat2_sub}"
brand = row.get("Product Family", "")
if brand:
    row["Category 2"] = f"Brands/{brand}"
# All products must appear in the catalog-wide "All Products" mega-menu category.
row["Category 3"] = "All Products"
if row.get("Variation Parent (StockKeepingUnit)"):
    row["Category 1"] = row["Category 2"] = row["Category 3"] = ""
```

## SKU naming convention

Adopt a deterministic, hierarchical pattern so SKUs are sortable, greppable, and tell you the parent/variant relationship at a glance:

```
[BRAND]-[CATEGORY]-[PRODUCT]-[VARIANT?]
```

- `[BRAND]` — short brand or family code, 3–4 chars (e.g. `BFG`, `VCE`, `VAL`).
- `[CATEGORY]` — short category code, 3–6 chars (e.g. `FERT`, `MEDIA`, `CONT`, `SEED`, `EXC` for excavator).
- `[PRODUCT]` — short product family identifier, free-form alphanumeric (e.g. `202020`, `PROMIXHP`, `EC220E`).
- `[VARIANT]` — only on variation children, encodes the variant axes joined by `-` (e.g. `1LB`, `5G-BLK`, `T235-HT`).

### Examples

| SKU | Decodes to |
|---|---|
| `BFG-FERT-202020` | BFG Supply, Fertilizer family, 20-20-20 — **VariationParent**, never purchased |
| `BFG-FERT-202020-1LB` | …same family, 1 lb variant |
| `BFG-FERT-202020-50LB` | …same family, 50 lb variant |
| `BFG-CONT-ROUND-5G-BLK` | Round nursery container, 5 gallon, black — two-axis variant |
| `VCE-EC220E` | Volvo CE Excavator EC220E — Simple product (no variations) |
| `VAL-T-SERIES` | Valtra T Series tractor — VariationParent |
| `VAL-T-235-HT` | …Valtra T 235 HP HiTech variant |

### Rules

- **Uppercase + dashes only.** No spaces, no underscores, no special chars. SKU goes into URLs (`/products/<sku>`) so keep it URL-safe.
- **Parent SKU = first three segments**; variant SKUs append a `-[VARIANT]` suffix to the parent. This makes `BaseSKU` introspectable: a child's `BaseSKU` always equals its parent's full SKU.
- **Length budget**: keep SKUs ≤ 30 chars where possible — the storefront SKU column is narrow.
- **Don't encode price tier or buyer group** in the SKU — those are pricebook concerns, not product identity.
- **Variant suffix is human-readable**, not a hash: `-1LB` over `-V01`. Demos sell on legibility.

### When deviating

If the user provides their own SKU scheme (e.g., already has SAP article numbers `100022345`), respect it — set `StockKeepingUnit` to their value but still derive `BaseSKU` for the parent grouping logic. Don't auto-rewrite real SKUs.

## Currency header convention

When currency ≠ USD, rename the three price columns:
- `Price (sale) EUR`, `Price (original) EUR`, `Price (VIP Pricing) EUR`

All three columns must share the same currency suffix.

## Optional 4th price tier

If the user requests `Special Pricing`, insert it between `Price (original)` and `Price (VIP Pricing)`:
- `Price (sale)`, `Price (original)`, `Price (Special Pricing)`, `Price (VIP Pricing)` — all in the same currency.

## Pricing rule for variation parents

**VariationParent rows must carry a normal/realistic price**, NOT a `0.01` placeholder. Use the price of the cheapest variant, or a representative "from" price.

Why this rule changed (2026-05-11, Dentaid demo):
- Setting parents to `0.01` made the PLP show literal `0,01 €` in every variation parent card — broken UX, looks like the storefront is giving products away free.
- The stock LWR `productCard` does NOT auto-detect `VariationParent` and show "From $X" using the cheapest variant. Parents render their own `PricebookEntry.UnitPrice` literally.
- Pricing engine still requires a `PricebookEntry` per product, so parents need a row — just with a sensible value.

Convention used by this skill:

| Row type | `Price (sale)` | `Price (original)` | `Price (VIP Pricing)` |
|---|---|---|---|
| Simple product | real | real | real |
| VariationParent | cheapest variant price (or representative "from" price) | cheapest variant list price | cheapest variant VIP price |
| Variation child | real | real | real |

The Advanced Import creates `PricebookEntry` rows from these columns. After the import, the orchestrator (or `sf-b2b-store-generator` Phase I) inserts `PricebookEntry` rows for the buyer-group pricebooks (List/Sale/VIP) using the same convention.

**Implementation hint for build script**: when generating the parent row, compute `min(child_prices)` per pricebook and use that. If the user supplies a price file with parents at `0.01`, override them to the cheapest variant's price and note the change in the Step 7 summary.

**Do NOT use `0.01` for parents.** This is a deprecated rule — old catalogs that still have it should be reimported with corrected parent prices.

## Quoting rules (CSV output)

- Use `csv.DictWriter(f, fieldnames=HEADER, quoting=csv.QUOTE_MINIMAL)`.
- Any field containing a comma, newline, or double-quote is auto-quoted.
- Product descriptions almost always contain commas — do not hand-format rows.

## Variation rows

A variation family is structured as:

**Parent row** (a real product):
- `SKU = VALTRA-T-SERIES` (example)
- `Variation AttributeSet = Valtra_T_Transmission` — only on the parent, names the attribute set
- All other variation columns blank
- Has its own prices and images

**Child rows** (one per variation):
- `SKU = VALTRA-T235-HT` — unique
- `Variation Parent (StockKeepingUnit) = VALTRA-T-SERIES` — points to parent
- `Variation Attribute Name 1 = Transmission`
- `Variation Attribute Value 1 = HiTech Powershift`
- Can set name/value 2 for a second axis (e.g. `Power = 235 HP`)
- Variation rows typically omit images and inherit from the parent — but can override.
- **Variations do NOT count against the 20-product target**.

## Worked example (single product, minimal)

```python
import csv

HEADER = [
    "Product Name", "Product Description", "SKU", "Product Family",
    "Category 1", "Category 2", "Category 3",
    "Price (sale) USD", "Price (original) USD", "Price (VIP Pricing) USD",
    "Entitlement 1", "Entitlement 2", "Entitlement 3", "Product isActive",
    "Variation AttributeSet",
    "Variation Attribute Name 1", "Variation Attribute Value 1",
    "Variation Attribute Name 2", "Variation Attribute Value 2",
    "Variation Parent (StockKeepingUnit)",
    "Media Standard Url 1", "Media Standard AltText 1",
    "Media Standard Url 2", "Media Standard Url 3",
    "Media Standard Url 4", "Media Standard Url 5",
    "Media Listing Url 1", "Media Listing AltText 1",
    "Media Attachment Url 1", "Media Attachment Url 2", "Media Attachment Url 3",
]

rows = [
    {
        "Product Name": "Volvo EC220E Crawler Excavator",
        "Product Description": "The Volvo EC220E is a 22-tonne class ...",  # 100-300 words
        "SKU": "VCE-EC220E",
        "Product Family": "Volvo CE",
        "Category 1": "Construction Equipment/Excavators",
        "Category 2": "Brands/Volvo CE",
        "Category 3": "All Products",
        "Price (sale) USD": "195000",
        "Price (original) USD": "215000",
        "Price (VIP Pricing) USD": "182000",
        "Entitlement 1": "Ascendum Entitlement Policy",
        "Product isActive": "TRUE",
        "Media Standard Url 1": "https://upload.wikimedia.org/wikipedia/commons/d/d7/Volvo_EC220DL.jpg",
        "Media Standard AltText 1": "Volvo EC220E Crawler Excavator",
        "Media Standard Url 2": "https://upload.wikimedia.org/wikipedia/commons/4/49/Volvo_EC220D_Dutch_Army.jpg",
    },
]

with open("out.csv", "w", newline="", encoding="utf-8") as fp:
    w = csv.DictWriter(fp, fieldnames=HEADER, quoting=csv.QUOTE_MINIMAL)
    w.writeheader()
    for r in rows:
        # Fill any missing columns with empty strings
        w.writerow({k: r.get(k, "") for k in HEADER})
```

## Validation snippet

After writing, always run:

```python
import csv
with open(path) as f:
    rows = list(csv.reader(f))
header = rows[0]
assert all(len(r) == len(header) for r in rows), "Column count mismatch"
print(f"{len(header)} columns, {len(rows)-1} data rows — OK")
```
