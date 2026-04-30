# Variations Metadata — The Full Recipe

This reference is how the skill creates the Salesforce metadata required for B2B Commerce variations. It's only used in Step 3b of the workflow, and only after the user has explicitly approved.

**Validated against Commerce Advanced on 2026-04-23.**

## The full model (all FIVE pieces required)

A working variation in B2B Commerce needs:

1. **Picklist custom fields on `ProductAttribute`** (one per attribute, e.g. `Transmission__c`) — metadata deploy.
2. **Field-Level Security (FLS)** granting read+edit to the fields on the permission sets the Commerce importer uses — **`B2B_Commerce_Cart_Upload`** is the critical one; also add `B2B_Commerce_Order_Builder` and `B2BCommerce_Community_Access`. **Without FLS on these, the importer returns `'<Field>__c' isn't a valid field for Product Attribute`** even though the field physically exists. The pre-seeded fields like `Size__c` work because they already have FLS on ~16 permission sets. Deploy a PermissionSet assigning FLS, and/or insert `FieldPermissions` records directly. See Step 3b below.
3. **`ProductAttributeSet` records** (one per variation family, e.g. `Valtra_T_Variations`) — data insert.
4. **`ProductAttributeSetItem` records** (one per (set × field) linkage) — data insert. Omitting these causes `The selected attribute set is empty`.
5. **CSV rows**:
   - Parent row: `Variation AttributeSet = <DeveloperName>` and populated `Category 1/2`
   - Child rows: `Variation Attribute Name N = <ApiName>__c`, `Variation Attribute Value N = <picklist value literal>`, `Variation Parent = <parent SKU>`

Omit any of the five and the importer fails.

### The FLS gotcha explained

The Commerce importer runs under system permissions plus the caller's FLS. A field can be fully deployed and visible via Tooling API but **invisible to the importer** if no Commerce permission set grants FLS. The error message `'<Field>__c' isn't a valid field for Product Attribute` is misleading — it should say "not readable". When the parent's children fail for this reason, the parent row fails too with `Choose a variation parent product` (cascade effect).

Symptoms to recognize this class of error:
- Fields exist in Tooling API (`SELECT FROM CustomField`).
- `ProductAttributeSetItem` records exist and reference the CustomField Id correctly.
- `SELECT ... FROM FieldPermissions WHERE Field = 'ProductAttribute.<Field>__c' AND PermissionsEdit = true` returns **< 5 records** — a red flag. Pre-seeded fields return 10-16 records.

## Step 1 — Gather what to create

From the catalog plan, collect all unique variation attributes and values. Example:

```python
variations = {
    "Valtra_T_Variations": {
        "parent_sku": "VALTRA-T-SERIES",
        "label": "Valtra T Variations",
        "attributes": {
            "Transmission": ["HiTech Powershift", "Versu Powershift", "Direct CVT"],
            "Power":        ["155 HP", "235 HP", "271 HP"],
        },
    },
    "Valtra_G125_Variations": {
        "parent_sku": "VALTRA-G125",
        "label": "Valtra G125 Variations",
        "attributes": {
            "Transmission": ["HiTech Powershift", "Versu Powershift", "Direct CVT"],
            "Power":        ["125 HP"],
        },
    },
}
```

**Merge attributes across families**: `Transmission` appears in both — deploy ONE `Transmission__c` field with the union of picklist values. The set-to-field linkage is done in Step 4 via `ProductAttributeSetItem`.

Map each attribute name to an API name: `Transmission` → `Transmission__c`. Sanitize: spaces → underscores, no accents, ≤ 40 chars.

**Idempotency check**:

```bash
sf data query --use-tooling-api --target-org <alias> \
  --query "SELECT Id, DeveloperName FROM CustomField WHERE TableEnumOrId='ProductAttribute' AND DeveloperName IN ('Transmission','Power')" \
  --json
```

Any field that already exists → skip its metadata deploy (but still reference it in Step 4). `ProductAttribute` in some orgs ships with Color / Size / Clothing_Sizes / Warranty_Options — reuse these if they fit the catalog's variations.

## Step 2 — Generate the metadata

```
metadata/
  sfdx-project.json
  package.xml
  force-app/main/default/
    objects/
      ProductAttribute/
        fields/
          Transmission__c.field-meta.xml
          Power__c.field-meta.xml
    cspTrustedSites/            # Step 2c output — deploy together
      Wikimedia_Commons.cspTrustedSite-meta.xml
```

### `sfdx-project.json`

```json
{
  "packageDirectories": [{ "path": "force-app", "default": true }],
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "62.0"
}
```

### Picklist field metadata

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Transmission__c</fullName>
    <label>Transmission</label>
    <type>Picklist</type>
    <required>false</required>
    <valueSet>
        <restricted>true</restricted>
        <valueSetDefinition>
            <sorted>false</sorted>
            <value><fullName>HiTech Powershift</fullName><default>false</default><label>HiTech Powershift</label></value>
            <value><fullName>Versu Powershift</fullName><default>false</default><label>Versu Powershift</label></value>
            <value><fullName>Direct CVT</fullName><default>false</default><label>Direct CVT</label></value>
        </valueSetDefinition>
    </valueSet>
</CustomField>
```

### `package.xml` (used for redeploys / deploy by manifest)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
  <types>
    <members>ProductAttribute.Transmission__c</members>
    <members>ProductAttribute.Power__c</members>
    <name>CustomField</name>
  </types>
  <types>
    <members>Wikimedia_Commons</members>
    <name>CspTrustedSite</name>
  </types>
  <version>62.0</version>
</Package>
```

Always maintain a `package.xml` — deploying `--source-dir` uses sfdx's local state tracking which cached "already deployed" on a re-run and blocked redeploys. `--manifest package.xml` is deterministic.

## Step 3 — Deploy

```bash
cd b2bCatalog/<Customer>/metadata
sf project deploy start --target-org <alias> --manifest package.xml --wait 10 --json
```

Parse: `result.status == "Succeeded"` AND `result.success == true`. `sf` prepends a yellow warning line to JSON output — strip lines before the first `{` (e.g. `| sed -n '/^{/,$p'`).

**File corruption gotcha**: don't `touch` the XML files to force a redeploy — it truncates them to 0 bytes on some shells. Write the XML content anew if you need to force re-upload.

**Describe cache gotcha**: `sf sobject describe --sobject ProductAttribute` caches for minutes after deploy. Verify with the **Tooling API** instead:

```bash
sf data query --use-tooling-api --target-org <alias> \
  --query "SELECT Id, DeveloperName, TableEnumOrId FROM CustomField WHERE DeveloperName IN ('Transmission','Power')" --json
```

The Id returned here is the **CustomField Id** (`00N...`) — you need it for Step 4.

## Step 3b — Grant FLS on the new custom fields

After deploying the custom fields (Step 3), add them to the Commerce permission sets. **Two complementary paths, do both**:

### Path A — Deploy a PermissionSet

```
force-app/main/default/permissionsets/
  <Customer>_Variation_Fields.permissionset-meta.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <label><Customer> Variation Fields</label>
    <description>FLS for custom Picklist fields used by the <Customer> B2B Commerce catalog variations.</description>
    <hasActivationRequired>false</hasActivationRequired>
    <fieldPermissions>
        <field>ProductAttribute.Transmission__c</field>
        <readable>true</readable>
        <editable>true</editable>
    </fieldPermissions>
    <fieldPermissions>
        <field>ProductAttribute.Power__c</field>
        <readable>true</readable>
        <editable>true</editable>
    </fieldPermissions>
</PermissionSet>
```

Add to `package.xml`:

```xml
<types>
  <members><Customer>_Variation_Fields</members>
  <name>PermissionSet</name>
</types>
```

Deploy, then assign to the user running the Commerce Import:

```bash
PSET_ID=$(sf data query --target-org <alias> --query "SELECT Id FROM PermissionSet WHERE Name='<Customer>_Variation_Fields'" --json | jq -r '.result.records[0].Id')
USER_ID=$(sf org display user --target-org <alias> --json | jq -r '.result.id')
sf data create record --sobject PermissionSetAssignment \
  --values "PermissionSetId='$PSET_ID' AssigneeId='$USER_ID'" \
  --target-org <alias> --json
```

### Path B — Insert FieldPermissions into existing Commerce permission sets

The importer may not run under your user — it runs under the Commerce integration user/profile. Mirror what `Size__c` / `Color__c` (pre-seeded, working) have by inserting `FieldPermissions` records into the key Commerce permission sets:

```bash
# Find the key Commerce permission sets
sf data query --target-org <alias> \
  --query "SELECT Parent.Name, ParentId FROM FieldPermissions WHERE Field = 'ProductAttribute.Size__c' AND PermissionsEdit = true" --json
```

Target at minimum:
- `B2B_Commerce_Cart_Upload` — **most important** — used by the CSV upload tool
- `B2B_Commerce_Order_Builder`
- `B2BCommerce_Community_Access`

For each (permission-set × field) combo:

```bash
sf data create record --sobject FieldPermissions \
  --values "ParentId='<psetId>' SObjectType='ProductAttribute' Field='ProductAttribute.Transmission__c' PermissionsRead=true PermissionsEdit=true" \
  --target-org <alias> --json
```

Use both Path A and Path B for maximum coverage — they're not redundant (Path A covers your admin user, Path B covers the import runtime user).

### Verify

```bash
sf data query --target-org <alias> \
  --query "SELECT Parent.Name FROM FieldPermissions WHERE Field = 'ProductAttribute.Transmission__c' AND PermissionsEdit = true" --json
```

Expect ≥ 4 permission sets. If < 3, the importer will still likely fail.

## Step 4 — Create ProductAttributeSet + ProductAttributeSetItem records

**This is the step that was missing from the original broken flow.**

### 4a — Create the AttributeSets

```bash
sf data create record --sobject ProductAttributeSet \
  --values "DeveloperName='Valtra_T_Variations' MasterLabel='Valtra T Variations' Description='Transmission and Power variants for Valtra T Series'" \
  --target-org <alias> --json
```

Required fields: `DeveloperName`, `MasterLabel`. Capture the returned `id` — you need it for 4b.

### 4b — Create the SetItems (one per set × field)

`ProductAttributeSetItem` has three required fields: `ProductAttributeSetId`, `Field`, `Sequence`.

**`Field` expects the CustomField Id**, NOT the API name string, NOT an AttributeDefinition Id. Using the API name fails with `Custom Field Definition ID: bad value for restricted picklist field: <Name>__c`. Using an AttributeDefinition Id fails with the same error. Use the `00N...` Id returned by the Tooling API query in Step 3.

```bash
sf data create record --sobject ProductAttributeSetItem \
  --values "ProductAttributeSetId='<setId>' Field='<customFieldId_00N...>' Sequence=1" \
  --target-org <alias> --json
```

Create one record per (attribute-set × custom-field) linkage. Sequence starts at 1 and increments per set.

For the Ascendum demo this was:
- Valtra_T_Variations × Transmission__c × seq 1
- Valtra_T_Variations × Power__c × seq 2
- Valtra_G125_Variations × Transmission__c × seq 1
- Valtra_G125_Variations × Power__c × seq 2

### 4c — Verify

```bash
sf data query --target-org <alias> \
  --query "SELECT ProductAttributeSet.DeveloperName, Field, Sequence FROM ProductAttributeSetItem ORDER BY ProductAttributeSet.DeveloperName, Sequence" --json
```

Should show every set with its items. If a set has no items, importer will reject with `The selected attribute set is empty`.

## Step 5 — Populate the CSV

After 1–4 succeed, update the CSV:

- **Parent row** (e.g. `SKU = VALTRA-T-SERIES`): `Variation AttributeSet = Valtra_T_Variations` (the DeveloperName)
- **Child row** (e.g. `SKU = VALTRA-T235-HT`):
  - `Variation Attribute Name 1 = Transmission__c`
  - `Variation Attribute Value 1 = HiTech Powershift`
  - `Variation Attribute Name 2 = Power__c`
  - `Variation Attribute Value 2 = 235 HP`
  - `Variation Parent (StockKeepingUnit) = VALTRA-T-SERIES`

**Cleanup rule**: if you drop a variation family (user said "no" or dropping children because values don't match picklists), **also clear the parent's `Variation AttributeSet`**. Orphan AttributeSet references on a parent with no child rows confuse the importer.

## Stale products — always check before reimporting a catalog with variations

A common pitfall: you imported the CSV once (without variations, or with broken variations), Salesforce created the products as `ProductClass = 'Simple'`. Now you reimport with variation rows — the importer finds the existing Simple product with the same SKU and refuses to promote it. Error: `Choose a variation parent product` on the parent rows, `Enter a value in the Variation Parent field only when...` on the children.

**Before reimporting with variations, check stale products**:

```bash
SKUS="'VALTRA-T-SERIES','VALTRA-G125'"   # just the SKUs that need to be VariationParent
sf data query --target-org <alias> \
  --query "SELECT Id, StockKeepingUnit, ProductClass FROM Product2 WHERE StockKeepingUnit IN ($SKUS)" --json
```

If any of them show `ProductClass = 'Simple'`, delete those Product2 records. The importer will recreate them as `VariationParent` because the new CSV has `Variation AttributeSet` populated on those rows.

```bash
sf data delete record --sobject Product2 --record-id <Id> --target-org <alias>
```

**Note**: you only need to delete parents that need to be promoted. Products that stay Simple (no variations) can be left alone — the importer will upsert them.

**Check also for related `ProductAttribute` records before deleting** — they have an FK and would block the delete:

```bash
sf data query --target-org <alias> \
  --query "SELECT Id FROM ProductAttribute WHERE ProductId IN ('<id1>','<id2>')" --json
```

Delete any `ProductAttribute` records first, then the parent `Product2`.

## Do NOT do this (lessons from 2026-04-23)

- ❌ Don't create Picklist fields on `Product2` — variation attributes live on `ProductAttribute`.
- ❌ Don't use `AttributeDefinition` / `AttributePicklist` / `AttributePicklistValue` — that's a different subsystem in Commerce. The B2B Advanced importer does not use them for variations.
- ❌ Don't use the API name (`Transmission__c`) in `ProductAttributeSetItem.Field` — it requires the CustomField Id.
- ❌ Don't skip `ProductAttributeSetItem` — without them, the importer rejects the set as empty.
- ❌ **Don't skip FLS**. Deploying the fields is not enough. Without FLS on `B2B_Commerce_Cart_Upload` (and friends), the importer fails with `'<Field>__c' isn't a valid field for Product Attribute`. Cascade effect: the parents also fail with `Choose a variation parent product`.
- ❌ **Don't reimport variations on top of existing Simple products**. The importer doesn't promote Simple → VariationParent. Delete the stale parents first.
- ❌ Don't `touch` XML files to force a redeploy — it may truncate them.
- ❌ Don't rely on `sf project deploy start --source-dir` after a first deploy — local state tracking blocks redeploys with "No local changes to deploy". Use `--manifest package.xml` instead.
- ❌ Don't include obsolete CspTrustedSite fields (`isApplicableToFrameAncestors`, etc.) — the deploy parser rejects them in current API versions.

## Error catalog

| Error | Cause | Fix |
|-------|-------|-----|
| `'X__c' isn't a valid field for Product Attribute` | Field exists but importer can't read it due to missing FLS | Add FLS on `B2B_Commerce_Cart_Upload` / `B2B_Commerce_Order_Builder` / `B2BCommerce_Community_Access`. See Step 3b. |
| `Choose a variation parent product` on the parent row | Cascade from variations failing, OR — most common — parent SKU already exists in the org with `ProductClass = 'Simple'` from a previous partial import. The importer will NOT promote a Simple to VariationParent. | Query `Product2` by the parent SKUs and check `ProductClass`. If Simple, delete them before reimport. See "Stale products" below. |
| `Enter a value in the Variation Parent field only when the value in the Product field isn't a variation parent` | The child row's parent SKU resolves to a `Simple` product (not `VariationParent`). Same root cause as above. | Delete the Simple product with that SKU; let the importer recreate it as VariationParent. |
| `The selected attribute set is empty` | `ProductAttributeSet` exists but has no `ProductAttributeSetItem` children | Run Step 4b |
| `Custom Field Definition ID: bad value for restricted picklist field` | Passing API name or AttrDef Id to `ProductAttributeSetItem.Field` | Use the CustomField `00N...` Id |
| `RequiresProjectError` | Not in sfdx project folder when running `sf project deploy` | `cd` into `b2bCatalog/<Customer>/metadata` |
| `NothingToDeploy: No local changes to deploy` | sfdx local state cache thinks nothing changed | Use `--manifest package.xml` instead of `--source-dir` |
| `error parsing field-meta.xml: Start tag expected` | File was truncated (e.g., by `touch`) | Rewrite the XML from scratch |
| `Element {...}isApplicableToFrameAncestors invalid` | CspTrustedSite XML has obsolete fields | Strip to `description / endpointUrl / isActive / context` only |
