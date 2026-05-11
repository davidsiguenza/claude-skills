# Demo Scenario Validation Checklist

**When to use**: after the Advanced Import succeeds and the smoke test (Step 7 #5) shows healthy counts, walk these scenarios end-to-end before declaring the demo ready. A catalog that imports cleanly can still be unusable on the storefront — these scenarios exercise the buyer-facing surfaces that demos actually show.

Each scenario has:
- **Setup**: prerequisites and where to navigate.
- **Walk**: step-by-step the operator does in the storefront.
- **Pass criteria**: explicit checks.
- **Common failures + diagnosis**: top 1–3 root causes with the SOQL/Apex to confirm.

The scenarios are ordered roughly by frequency — Scenario 1 is the most common demo opener, Scenario 5 is advanced. The BFG Supply demo pattern is a useful benchmark for a presentable catalog: variation selectors, attribute-driven faceted search, buyer-specific pricing, and reorder are the four flows that make the demo feel like real B2B Commerce instead of a product list.

---

## Scenario 1 — Variant selector on the PDP

**Demonstrates**: variation parent → child relationship surfaces correctly; price updates per variant; Add to Cart gates on selection.

### Setup
- A `VariationParent` product imported with at least 2 `Variation` children.
- Both pricebooks (Standard + Buyer Group) have `PricebookEntry` rows for every child.
- The buyer logged in is a member of the buyer group whose pricebook is active.

### Walk
1. Navigate to the parent product's category (e.g., `Grower > Fertilizers`).
2. Click the parent product (e.g., `20-20-20 All Purpose Fertilizer`).
3. Confirm the variant selector renders with the **variant selector field(s)** of the attribute set (e.g., `Package Size`: 1 lb / 5 lb / 25 lb / 50 lb).
4. Click each variant option in turn.

### Pass criteria
- All variant options appear in the selector.
- Price updates each time you click a different variant.
- `Add to Cart` is **disabled** before any variant is selected, and **enables** the moment you pick one.
- Image optionally swaps if the variant has its own image (otherwise inherits parent's).

### Common failures
| Symptom | Likely cause | Diagnose |
|---|---|---|
| Variant selector shows nothing / single button | `ProductAttributeSet` exists but no `ProductAttributeSetItem` records linking it to fields | `SELECT ProductAttributeSet.DeveloperName, Field FROM ProductAttributeSetItem` — empty? Run VARIATIONS_METADATA.md Step 4b. |
| Variant selector shows but wrong attribute | Wrong field linked in `ProductAttributeSetItem` (or wrong CustomField Id) | Same query as above — verify the `Field` column matches the CustomField `00N...` Id of the intended attribute. |
| Variant selector shows but `Add to Cart` never enables | `ProductVariantAttribute` records missing on children | `SELECT COUNT() FROM ProductVariantAttribute WHERE ProductId IN (<child Ids>)` — should be ≥ 1 per child per axis. The Advanced Import creates these automatically from `Variation Attribute Name N / Value N` columns; missing means those columns were blank in the CSV. |
| Price doesn't update on variant click | Child has no `PricebookEntry` in the buyer's active pricebook | `SELECT Pricebook2.Name, UnitPrice FROM PricebookEntry WHERE Product2Id = '<childId>' AND IsActive = true` — confirm the buyer-group pricebook appears. If not, run the post-import pricebook seeding step. |

---

## Scenario 2 — Faceted filtering on the PLP

**Demonstrates**: search index has indexed the facet fields; the storefront's left-rail filter UX works.

### Setup
- A category with ≥ 5 products that share an attribute set.
- Search index rebuilt **after** the product import (otherwise products are not indexed).
- Site published after the rebuild.

### Walk
1. Navigate to the category page (e.g., `Grower > Fertilizers & Plant Nutrition`).
2. Confirm the left-rail filter section renders with the **facet fields** of the attribute set (e.g., `Formulation Type`, `Application Method`, `NPK Ratio`).
3. Click one filter value (e.g., `Liquid Concentrate` under Formulation Type).
4. Confirm the product grid narrows to only products with that attribute value.
5. Click a second filter (e.g., `Foliar Spray` under Application Method) — should narrow further (AND semantics).
6. Click `Clear all filters` — full grid returns.

### Pass criteria
- Left-rail facets render with all expected attributes.
- Each filter shows accurate counts (e.g., `Liquid Concentrate (3)`).
- Filtering narrows the product grid correctly.
- Clearing filters restores the full grid.

### Common failures
| Symptom | Likely cause | Diagnose |
|---|---|---|
| No facets in the left rail | Search index not rebuilt after import OR fields not marked as `Searchable` | Run `sf commerce search start --store-name '<Site Name>' --target-org <alias>`, wait ~2 min, and refresh. If still missing, check `Setup → Commerce → Stores → <store> → Search → Searchable Fields`. |
| Some facets missing, others appear | The missing field doesn't have any picklist value populated on any product | `SELECT COUNT() FROM ProductAttribute WHERE <Field>__c != null` — if 0, the importer didn't write to that field (FLS issue or column missing in CSV). |
| Filter shows but produces 0 results | Picklist values in CSV don't match the deployed picklist values exactly (case-sensitive, including punctuation) | Compare `SELECT <Field>__c FROM ProductAttribute LIMIT 10` to the deployed picklist's API names. |
| Counts shown but stale (e.g., (0) on every value) | Search index ran during import and didn't reindex products created later | Rebuild again. The index can race with import. |

---

## Scenario 3 — Buyer-specific pricing

**Demonstrates**: the same product shows different prices for different buyers based on buyer group membership.

### Setup
- At least 2 buyers exist: one in the **VIP buyer group** (or "Sale" group), one in the **Standard buyer group**.
- All custom pricebooks (List/Sale/VIP) have `PricebookEntry` rows for every active product.
- `BuyerGroupPricebook` records link each pricebook to its buyer group with `IsActive = true`.

### Walk
1. **Open an incognito/private window**. Log in as the VIP buyer.
2. Navigate to a product (e.g., `BFG-FERT-202020-50LB`).
3. Note the displayed price.
4. Log out. **Open a second incognito window**. Log in as the Standard buyer.
5. Navigate to the same product. The price should be different (typically higher).
6. Optional: open a third window as guest (no login). If guest browsing is enabled, the price should be the List price.

### Pass criteria
- VIP buyer sees the lowest price.
- Standard buyer sees a higher price.
- Guest (if enabled) sees List price (or strike-through with Sale price).
- No buyer ever sees `Price Unavailable` on an active product.

### Common failures
| Symptom | Likely cause | Diagnose |
|---|---|---|
| Both buyers see the same price | One of the buyers is not a `BuyerGroupMember` of the expected group, OR the buyer-group pricebook is missing the variant | `SELECT BuyerGroup.Name FROM BuyerGroupMember WHERE BuyerId = '<accountId>'` for both. Then `SELECT Pricebook2.Name FROM PricebookEntry WHERE Product2Id = '<id>'` — confirm both pricebooks appear. |
| Some products show `Price Unavailable` | Variant has `PricebookEntry` only in the Standard pricebook, not the buyer-group one | `SELECT Pricebook2.Name FROM PricebookEntry WHERE Product2Id = '<id>' AND IsActive = true` — fix by inserting the missing rows. |
| Prices visible but stale (cache) | Pricing engine cached pre-update prices | `GET /services/data/v66.0/connect/core-pricing/sync/syncData` to flush. Or wait ~5 min. |
| `Price Unavailable` on every product | Currency mismatch: pricebook `CurrencyIsoCode` ≠ store/buyer currency | `SELECT Name, CurrencyIsoCode FROM Pricebook2 WHERE Id IN (...)` — confirm all match. |

---

## Scenario 4 — Reorder workflow

**Demonstrates**: a returning buyer can repeat a previous order with one click. Tests the order history and account-context UX.

### Setup
- The buyer has at least 1 prior `Order` (placed via storefront checkout, not just `INSERT INTO Order`).
- The order's `OrderItem` rows reference active products that still exist.
- The buyer's `Account` has a primary `Contact` linked to the user.

### Walk
1. Log in as the buyer.
2. Navigate to **My Account → Order History** (or similar — exact path varies by template).
3. Click a recent order.
4. Click **Reorder** (or **Buy Again**).
5. Confirm the cart is populated with the same SKUs and quantities.
6. Adjust if desired and proceed to checkout.

### Pass criteria
- Order History shows the prior order with correct totals and date.
- Reorder button populates the cart correctly.
- Quantities match the original order.
- Discontinued/inactive products are flagged but don't break the flow.

### Common failures
| Symptom | Likely cause | Diagnose |
|---|---|---|
| Order History page is empty for this buyer | Buyer's `Contact` not linked to the `Account` correctly, OR orders were placed under a different user | `SELECT AccountId, OwnerId FROM Order WHERE OwnerId = '<userId>'` — ensure the buyer's user owns at least one order. |
| Reorder button missing | Storefront template doesn't have the Reorder LWC enabled, OR Connect API order endpoint disabled | Check Experience Builder → component palette → enable Reorder LWC on the OrderConfirmation/OrderHistory page. |
| Reorder populates 0 items | Original order's products are now `IsActive = false` or deleted | `SELECT Product2.Name, Product2.IsActive FROM OrderItem WHERE OrderId = '<id>'` — reactivate or remove. |

---

## Scenario 5 — Multi-axis variant (advanced)

**Demonstrates**: two-attribute variation selectors (e.g., Size × Color) work end-to-end. This scenario fails the most often — only run if the catalog has two-axis variations.

### Setup
- A `VariationParent` whose `ProductAttributeSet` has **2 ProductAttributeSetItem records** (e.g., `Container_Specification` with `Container_Size__c` + `Color__c`).
- Children must populate **both** `Variation Attribute Name 1/Value 1` AND `Variation Attribute Name 2/Value 2` on every variant row in the CSV (no blanks).
- A coherent matrix of children: e.g., `1 gal × Black`, `1 gal × Green`, `5 gal × Black`, `5 gal × Green` — for two axes with N×M children. Demos break if you have e.g. `1 gal × Black` and `5 gal × Green` only — the buyer can pick `1 gal × Green` and see "out of stock" or no variant matched.

### Walk
1. Navigate to a multi-axis parent (e.g., `Round Nursery Container`).
2. Confirm the variant selector shows **two dropdowns/swatches**: one per axis.
3. Click axis 1 (e.g., `Container Size = 1 gal`). Axis 2 should still allow all valid colors.
4. Click axis 2 (e.g., `Color = Black`). Both axes should now be set.
5. Confirm price updates and Add to Cart enables.
6. Click axis 1 → `5 gal`. Axis 2 should re-resolve (or stay on Black if 5 gal × Black exists).
7. Try a combination that doesn't exist — e.g., `15 gal × Terra Cotta`. Should be unselectable or show "not available".

### Pass criteria
- Both axes render on the PDP variant selector.
- Selecting one narrows the other (if any combinations are missing).
- Add to Cart only enables when both axes are picked.
- Price reflects the specific combination, not the parent or the average.

### Common failures
| Symptom | Likely cause | Diagnose |
|---|---|---|
| Only one axis renders | Children only populate `Variation Attribute Name 1/Value 1`, `Name 2/Value 2` is blank | `SELECT COUNT() FROM ProductVariantAttribute WHERE ProductId = '<childId>'` — should be 2 per child for a 2-axis set. If 1, fix the CSV and reimport. |
| Both axes render but Add to Cart never enables | Children created with `ProductClass = 'Simple'` instead of `Variation` (importer didn't recognize them as variants) | `SELECT ProductClass FROM Product2 WHERE StockKeepingUnit IN (<child SKUs>)` — must be `Variation`. If Simple, see VARIATIONS_METADATA.md "Stale products". |
| Some combinations missing | CSV doesn't have the full N×M matrix | Build a coverage table from the CSV: for each axis-1 value, for each axis-2 value, assert there's a child row. Demos work better with a complete matrix even if a few are "discontinued". |
| Wrong price on a combo | One child has the wrong `PricebookEntry` | `SELECT StockKeepingUnit, UnitPrice FROM PricebookEntry WHERE Product2.StockKeepingUnit IN (<combo SKUs>)` — fix outliers. |

---

## Quick failure → scenario lookup

| You see this on the storefront | Run this scenario |
|---|---|
| Products in category but no filters render | Scenario 2 |
| Click product → no variant selector | Scenario 1 |
| Same price for everyone | Scenario 3 |
| `Price Unavailable` everywhere | Scenario 3 |
| Can't repeat a prior order | Scenario 4 |
| 2-axis variations broken | Scenario 5 |

## BFG-style presentability checklist

Use this as a final operator checklist for horticulture, agriculture, industrial supply, or any catalog where the demo value comes from variants and faceted search:

1. **Category journey**: open each top-level category and one subcategory. Confirm products render, images appear, and the category path makes sense to a buyer.
2. **Variant journey**: open at least one parent product from every attribute-set family in the catalog. Confirm the selector labels match buyer language (`Package Size`, `Volume`, `Color`, etc.), not internal field names.
3. **Facet journey**: in the strongest category, apply one product-type facet and one spec facet. Confirm counts narrow correctly and clearing filters restores the grid.
4. **Buyer pricing journey**: compare the same SKU as guest, standard buyer, and elevated/VIP buyer when those personas exist. Confirm no persona sees `Price Unavailable`.
5. **Reorder journey**: place one small order, return to order history, and confirm reorder / buy-again can repopulate the cart.
6. **Image journey**: confirm both PLP/list images and PDP/detail images render. If internal Salesforce UI shows images but the storefront does not, diagnose CSP, CMS channel link, and CMS language in that order.

## Always-run smoke test

Before walking any scenario:

```bash
sf apex run --target-org <alias> --file /tmp/smokeTest.apex
```

(See SKILL.md Step 7 #5 for the script.) If any counter is 0, fix that root cause first — running scenarios on a partial import wastes time.
