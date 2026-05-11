# Attribute Set Templates — Reusable Library

A catalog of pre-designed `ProductAttributeSet` definitions that you can drop into Step 3b of the workflow when the catalog's vertical matches.

## ⚠️ Single-attribute set bias (read first)

The CSV Advanced Import requires **every attribute in a `ProductAttributeSet` to have a value for every variant**. Multi-attribute sets only work cleanly when every variant truly varies on every axis. If even one variant has a blank for one axis, the importer fails with `"Provide a value for each variation attribute field"` — and the failure cascades to the parent (`"Choose a variation parent product"`).

**Default to single-attribute sets** unless the catalog clearly needs multi-axis. Examples:

- A 12-variant grid `3 colors × 4 sizes` where every (color, size) pair exists → ONE 2-attribute set is correct.
- A catalog where some products vary only on color and others only on size → TWO 1-attribute sets, not one 2-attribute set with blanks.
- A heavy-equipment catalog where one model has Transmission options and another has Power options but not both → split into two single-attr sets.

The templates in this file annotate which are naturally multi-axis (Containers — size + color, Apparel — size + color, Heavy Equipment — transmission + power) — for those, only use the multi-attr set if every variant has both axes populated. For everything else, default to single-attr.

**How to use this file**:
1. In Step 3b, after deciding the catalog has variations, scan this file for a vertical that matches.
2. If a template matches, **propose its fields and picklist values to the user verbatim**, ask which `ProductAttribute` fields already exist on their org (idempotency check via Tooling API), and only deploy the missing ones.
3. If no template fits, design the set ad-hoc — **and add the new template here at the end of the workflow** so the next demo benefits.

For each template:
- **Variant selector field(s)** = the field(s) the buyer picks on the PDP (`Package_Size__c`, `Color__c`, etc.). Show on the PDP variant selector.
- **Facet fields** = filterable attributes on the PLP/category page. Don't show on the variant selector — they're properties of the parent, not the variant.
- All fields are Picklist, `<restricted>true</restricted>`. Deploy on `ProductAttribute` (not `Product2`).
- FLS is mandatory on `B2B_Commerce_Cart_Upload`, `B2B_Commerce_Order_Builder`, `B2BCommerce_Community_Access` — see VARIATIONS_METADATA.md Step 3b.

**BFG Supply lesson**: a strong B2B demo uses attribute sets for both PDP selection and PLP faceting. When adapting these templates, keep at least one buyer-visible selector field and 2-4 useful facet fields per major category. Avoid creating attributes that will never be populated in the CSV; empty facet fields do not help the storefront and can break variation imports when included in a variation set.

---

## Verticals covered

| # | Template | Verticals | Variant selector | Source |
|---|---|---|---|---|
| 1 | `Fertilizer_Formulation` | Fertilizers, plant nutrition, hydroponic nutrients | `Package_Size__c` | BFG Supply 2026-05 |
| 2 | `Growing_Media` | Soil, peat, coir, soilless mixes | `Package_Unit__c` | BFG Supply 2026-05 |
| 3 | `Container_Specification` | Pots, trays, cell packs, hanging baskets | `Container_Size__c` + `Color__c` | BFG Supply 2026-05 |
| 4 | `Seed_Specification` | Vegetable seed, herb seed, plugs | `Packet_Size__c` + `Variety_Color__c` | BFG Supply 2026-05 |
| 5 | `Pest_Disease_Control` | Insecticide, fungicide, herbicide | `Package_Size__c` | BFG Supply 2026-05 |
| 6 | `Hydroponic_System` | pH solutions, multi-part nutrients | `Volume__c` | BFG Supply 2026-05 |
| 7 | `Grass_Seed_Blend` | Grass seed, cover crops | `Bag_Size__c` | BFG Supply 2026-05 |
| 8 | `Heavy_Equipment_Variations` | Construction, agricultural machinery | `Transmission__c` + `Power__c` | Ascendum/Valtra 2026-04 |
| 9 | `Apparel_Sizing` | Fashion, workwear, uniforms | `Size__c` + `Color__c` | Pre-seeded org fields |

---

## Template 1 — `Fertilizer_Formulation`

**Verticals**: agricultural fertilizers, plant nutrition, hydroponic dry/liquid nutrients.
**Variant selector**: `Package_Size__c`
**Facet fields**: `Formulation_Type__c`, `Application_Method__c`, `NPK_Ratio__c`, `Nutrient_Focus__c`

| Field | Label | Picklist values |
|---|---|---|
| `Package_Size__c` | Package Size | `1 lb`, `5 lb`, `25 lb`, `50 lb`, `2.5 gal`, `5 gal`, `30 gal`, `55 gal` |
| `Formulation_Type__c` | Formulation Type | `Granular`, `Soluble Powder`, `Liquid Concentrate`, `Slow-Release`, `Controlled-Release` |
| `Application_Method__c` | Application Method | `Broadcast`, `Drench`, `Foliar Spray`, `Fertigation`, `Injector` |
| `NPK_Ratio__c` | NPK Ratio | `20-20-20`, `17-5-24`, `15-5-15`, `13-2-13`, `10-52-10`, `0-0-50`, `Custom` |
| `Nutrient_Focus__c` | Nutrient Focus | `Balanced`, `Bloom Booster`, `Root Development`, `Micronutrients`, `Calcium-Magnesium` |

---

## Template 2 — `Growing_Media`

**Verticals**: soil, peat-based mixes, coir, rockwool, perlite, soilless growing media.
**Variant selector**: `Package_Unit__c`
**Facet fields**: `Media_Type__c`, `pH_Range__c`, `Perlite_Content__c`, `Wetting_Agent__c`

| Field | Label | Picklist values |
|---|---|---|
| `Package_Unit__c` | Package Unit | `1 cu ft bag`, `2 cu ft bag`, `3.8 cu ft bale`, `Pallet (60 bags)`, `Pallet (30 bales)` |
| `Media_Type__c` | Media Type | `Peat-Based`, `Coir-Based`, `Bark-Based`, `Rockwool`, `Perlite`, `Vermiculite`, `Blended Mix` |
| `pH_Range__c` | pH Range | `5.5–6.0`, `6.0–6.5`, `6.0–7.0`, `Unadjusted` |
| `Perlite_Content__c` | Perlite Content | `None`, `10%`, `20%`, `30%`, `40%+` |
| `Wetting_Agent__c` | Wetting Agent Included | `Yes`, `No` |

---

## Template 3 — `Container_Specification`

**Verticals**: nursery containers, trays, cell packs, hanging baskets, propagation supplies.
**Variant selector** (two-axis): `Container_Size__c` + `Color__c`
**Facet fields**: `Material__c`, `Container_Style__c`, `Pack_Quantity__c`

| Field | Label | Picklist values |
|---|---|---|
| `Container_Size__c` | Container Size | `2.5"`, `4"`, `6"`, `8"`, `1 gal`, `2 gal`, `3 gal`, `5 gal`, `7 gal`, `10 gal`, `15 gal`, `25 gal` |
| `Color__c` | Color | `Black`, `Azalea Green`, `Terra Cotta`, `White`, `Nursery Red`, `Standard Brown` |
| `Material__c` | Material | `Polypropylene`, `Polyethylene`, `Biodegradable`, `Fabric`, `Foam` |
| `Container_Style__c` | Container Style | `Round`, `Square`, `Azalea`, `Hanging Basket`, `Tray Insert`, `Flat Tray`, `Cell Pack` |
| `Pack_Quantity__c` | Pack Quantity | `Each`, `Sleeve of 10`, `Case of 50`, `Case of 100`, `Case of 250`, `Pallet` |

**Note**: `Color__c` is often pre-seeded on `ProductAttribute` in modern SDOs. Check before deploying — reuse the existing field and only add missing picklist values.

---

## Template 4 — `Seed_Specification`

**Verticals**: vegetable seed, herb seed, ornamental plugs.
**Variant selector** (two-axis): `Packet_Size__c` + `Variety_Color__c`
**Facet fields**: `Seed_Treatment__c`, `Days_to_Maturity__c`, `Growing_Season__c`

| Field | Label | Picklist values |
|---|---|---|
| `Packet_Size__c` | Packet Size | `100 seeds`, `250 seeds`, `500 seeds`, `1,000 seeds`, `5,000 seeds`, `1 oz`, `4 oz`, `1 lb` |
| `Variety_Color__c` | Variety / Color | `Mixed`, `Red`, `Yellow`, `Orange`, `White`, `Purple`, `Pink`, `Bicolor`, `Green` |
| `Seed_Treatment__c` | Seed Treatment | `Untreated`, `Pelleted`, `Primed`, `Coated`, `Organic Certified` |
| `Days_to_Maturity__c` | Days to Maturity | `Under 50 days`, `50–70 days`, `70–90 days`, `90+ days` |
| `Growing_Season__c` | Growing Season | `Cool Season`, `Warm Season`, `Year-Round`, `Perennial` |

---

## Template 5 — `Pest_Disease_Control`

**Verticals**: insecticide, fungicide, herbicide, miticide, bactericide.
**Variant selector**: `Package_Size__c`
**Facet fields**: `Product_Type__c`, `Organic_Status__c`, `Target_Pest__c`

| Field | Label | Picklist values |
|---|---|---|
| `Package_Size__c` | Package Size | `1 qt`, `1 gal`, `2.5 gal`, `5 gal`, `1 lb`, `5 lb`, `25 lb` |
| `Product_Type__c` | Product Type | `Insecticide`, `Fungicide`, `Herbicide`, `Miticide`, `Bactericide`, `Growth Regulator`, `Adjuvant` |
| `Organic_Status__c` | Organic Status | `OMRI Listed`, `EPA Minimum Risk`, `Conventional`, `Restricted Use` |
| `Target_Pest__c` | Target Pest | `Aphids`, `Whitefly`, `Spider Mites`, `Thrips`, `Fungus Gnats`, `Botrytis`, `Powdery Mildew`, `Broad Spectrum` |

**Field reuse**: `Package_Size__c` is shared with Template 1 — deploy ONE field with the union of picklist values, link via different `ProductAttributeSetItem` records to each set.

---

## Template 6 — `Hydroponic_System`

**Verticals**: pH solutions, multi-part nutrient lines, grow stage formulas.
**Variant selector**: `Volume__c`
**Facet fields**: `Nutrient_Part__c`, `Growing_Stage__c`, `Mineral_Organic__c`

| Field | Label | Picklist values |
|---|---|---|
| `Volume__c` | Volume | `1 L`, `4 L`, `10 L`, `20 L`, `208 L (drum)`, `1,000 L (tote)` |
| `Nutrient_Part__c` | Nutrient Part | `Part A`, `Part B`, `Part C`, `1-Part`, `2-Part`, `3-Part`, `pH Up`, `pH Down` |
| `Growing_Stage__c` | Growing Stage | `Seedling`, `Vegetative`, `Bloom`, `Fruiting`, `Clone`, `All Stages` |
| `Mineral_Organic__c` | Mineral / Organic | `Mineral (Synthetic)`, `Organic`, `Hybrid` |

---

## Template 7 — `Grass_Seed_Blend`

**Verticals**: turf grass seed, cover crops, pasture seed.
**Variant selector**: `Bag_Size__c`
**Facet fields**: `Grass_Type__c`, `Sun_Shade__c`, `Use_Type__c`

| Field | Label | Picklist values |
|---|---|---|
| `Bag_Size__c` | Bag Size | `3 lb`, `5 lb`, `10 lb`, `25 lb`, `50 lb` |
| `Grass_Type__c` | Grass Type | `Kentucky Bluegrass`, `Perennial Ryegrass`, `Tall Fescue`, `Fine Fescue`, `Bermuda`, `Zoysia`, `Bentgrass`, `Mixed Blend` |
| `Sun_Shade__c` | Sun / Shade | `Full Sun`, `Sun & Shade`, `Dense Shade`, `Partial Shade` |
| `Use_Type__c` | Use Type | `Athletic Field`, `Golf Course`, `Home Lawn`, `Overseeding`, `Erosion Control`, `Pasture` |

---

## Template 8 — `Heavy_Equipment_Variations`

**Verticals**: construction equipment, agricultural machinery, industrial vehicles.
**Variant selector** (two-axis): `Transmission__c` + `Power__c`
**Facet fields**: usually none — heavy equipment buyers pick exact specs, not filter.

| Field | Label | Picklist values |
|---|---|---|
| `Transmission__c` | Transmission | `HiTech Powershift`, `Versu Powershift`, `Direct CVT`, `Manual`, `Automatic` |
| `Power__c` | Power | `125 HP`, `155 HP`, `185 HP`, `235 HP`, `271 HP`, `Custom` |

**Note**: when sharing `Transmission__c` across multiple equipment lines (e.g., Valtra T + Valtra G125), deploy ONE field with the **union** of picklist values. Each line gets its own `ProductAttributeSet`, and `ProductAttributeSetItem` links the same field to multiple sets. Don't deploy `T_Transmission__c` and `G125_Transmission__c` separately.

---

## Template 9 — `Apparel_Sizing`

**Verticals**: clothing, workwear, uniforms, footwear.
**Variant selector** (two-axis): `Size__c` + `Color__c`
**Facet fields**: `Material__c`, `Style__c`, `Gender__c`

| Field | Label | Picklist values |
|---|---|---|
| `Size__c` | Size | `XS`, `S`, `M`, `L`, `XL`, `XXL`, `XXXL`, `One Size` |
| `Color__c` | Color | (catalog-specific — propose to user) |
| `Material__c` | Material | `Cotton`, `Polyester`, `Cotton/Poly Blend`, `Wool`, `Synthetic`, `Leather` |
| `Style__c` | Style | `Crew Neck`, `V-Neck`, `Polo`, `Button-Up`, `Hoodie`, `Long Sleeve`, `Short Sleeve` |
| `Gender__c` | Gender | `Men's`, `Women's`, `Unisex`, `Youth` |

**Critical**: `Size__c` and `Color__c` are pre-seeded on `ProductAttribute` in most SDOs — do NOT redeploy. Reuse the existing fields and link them to your custom `ProductAttributeSet`. The pre-seeded fields ship with FLS on ~16 permission sets, which is exactly what you want.

---

## Adding a new template

When the agent designs a new attribute set ad-hoc that doesn't match any of the above:

1. Complete the catalog flow.
2. Append a new section at the end of this file with the same structure: vertical, variant selector(s), facet fields, table of (Field × Label × Picklist values), notes on reuse with existing templates.
3. Update the "Verticals covered" table at the top.
4. Increment the template number.

The next demo in the same vertical reuses the template, saving the design conversation.

## Field-reuse matrix (idempotency cheatsheet)

When designing a new catalog, check first whether these common fields already exist on the org:

| Field | Often pre-seeded? | Templates that use it |
|---|---|---|
| `Color__c` | ✅ Most SDOs | 3 (Containers), 9 (Apparel) |
| `Size__c` | ✅ Most SDOs | 9 (Apparel) |
| `Material__c` | ⚠️ Sometimes | 3 (Containers), 9 (Apparel) |
| `Package_Size__c` | ❌ Custom | 1 (Fertilizer), 5 (Pest/Disease) |
| `Volume__c` | ❌ Custom | 6 (Hydroponic) |

Run before deploying:

```bash
sf data query --use-tooling-api --target-org <alias> \
  --query "SELECT Id, DeveloperName FROM CustomField WHERE TableEnumOrId = 'ProductAttribute' AND DeveloperName IN ('Color','Size','Material','Package_Size','Volume')" --json
```

Reuse what's there. Only deploy new fields for what's missing.
