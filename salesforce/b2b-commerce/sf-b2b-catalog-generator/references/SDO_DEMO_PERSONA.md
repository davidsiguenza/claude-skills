# SDO Standard Demo Persona — Lauren Bailey / Omega, Inc.

**Use this file when the catalog targets a B2B Commerce Enhanced SDO and the user has not asked for a custom buyer**. The persona below is pre-seeded in every B2B Commerce Enhanced SDO and is what most internal Salesforce demos use. Reusing it saves ~30 minutes of buyer-group / pricebook setup per demo.

> **Decision rule for the agent**: when running standalone (no storefront created in the same session), ask the user *"Use the standard SDO buyer persona (Lauren Bailey / Omega Inc.) or create custom buyers?"* — and default to the standard if the user says yes or doesn't have a strong preference. If `sf-b2b-store-generator` ran in the same session and created custom buyers, use those instead and ignore this file.

> **Package skill modes**: `sf-b2b-demo-builder` uses this persona automatically for Path B (**strict seeded SDO catalog demo**) and Path C (**hybrid branded seeded SDO demo**). Path B leaves the existing store branding untouched. Path C may rebrand the existing site's visual shell, but it still uses Lauren/Omega/Cirrus for buyer access, entitlement, and pricing.

## The persona

| Field | Value |
|---|---|
| **Contact** | Lauren Bailey |
| **Email** | `lbailey@example.com` |
| **Account** | Omega, Inc. |
| **Default Store** | `SDO - B2B Commerce Enhanced` |

## Buyer Groups (via `BuyerGroupMember` linkage to the Account)

| Buyer Group | Role | Pricebook |
|---|---|---|
| **Cirrus Buyer Group** | Standard buyer access | `Cirrus Price Book` |
| **Cirrus Silver Buyer Group** | Elevated tier — premium / account-specific pricing demos | `Cirrus Silver Price Book` |
| **D2C Commerce Buyer Group** | D2C storefront access | (D2C pricebook) |
| **Reorder Portal Buyer Group** | Reorder portal access | — |

The Account is a member of **all four** buyer groups, so the same buyer can demonstrate baseline pricing AND tiered/elevated pricing depending on which buyer group the demo highlights.

## Entitlement Policy

**`Cirrus Entitlement Policy`** — applies to both Cirrus and Cirrus Silver buyer groups. Use this as `Entitlement 1` in every CSV row when targeting the standard persona.

## Resolving IDs

```bash
# Cirrus Price Book (standard buyer)
sf data query --target-org <alias> \
  --query "SELECT Id FROM Pricebook2 WHERE Name = 'Cirrus Price Book'" --json

# Cirrus Silver Price Book (elevated tier)
sf data query --target-org <alias> \
  --query "SELECT Id FROM Pricebook2 WHERE Name = 'Cirrus Silver Price Book'" --json

# Lauren's Account
sf data query --target-org <alias> \
  --query "SELECT Id, Name FROM Account WHERE Name = 'Omega, Inc.'" --json

# Buyer groups for the account
sf data query --target-org <alias> \
  --query "SELECT BuyerGroup.Name, BuyerGroupId FROM BuyerGroupMember WHERE Buyer.Name = 'Omega, Inc.'" --json
```

## Pricing rule when using this persona

In B2B Commerce, **pricing is determined by buyer group pricebooks, NOT by the WebStore's pricebook**. So when targeting Lauren Bailey:

- Add every product to **Standard Pricebook** (platform requirement — source of truth).
- Add every product to **Cirrus Price Book** (so the standard demo buyer sees prices).
- Add every product to **Cirrus Silver Price Book** if the demo shows tiered pricing.

Skip any of these and Lauren sees `Pricing unavailable` on the storefront, **even though admin sees prices fine**.

After bulk pricebook insertions, **run the pricing sync API** (`GET /connect/core-pricing/sync/syncData`) to refresh the lookup tables — the most commonly forgotten step. See `SKILL.md` Step 7 #6.

## CSV mapping when using this persona

In `pricebookAliasToIdMapping` for the import job (see CSV_IMPORT_API.md Step 3):

```json
{
  "sale": "<Cirrus Price Book Id>",
  "original": "<Standard Pricebook Id>",
  "VIP Pricing": "<Cirrus Silver Price Book Id>"
}
```

Then the CSV's price columns become:

- `Price (sale) USD` → renders as the buyer's default price
- `Price (original) USD` → strikethrough / list price
- `Price (VIP Pricing) USD` → only visible to Cirrus Silver members

## Diagnose pricing for Lauren

If pricing doesn't show on the storefront when logged in as Lauren, run the diagnostic in DEMO_SCENARIOS.md Scenario 3 — the typical fix is "the variant is missing from one of Lauren's buyer-group pricebooks" + "forgot to run the pricing sync".

## When NOT to use this persona

- If `sf-b2b-store-generator` ran in the same session and created custom buyer groups (e.g., `BFG Grower Buyers`, `Ascendum VIP`, etc.) — use those instead. The store-generator's `branding-work/<slug>/store-ids.json` records which buyer groups it created.
- If the demo is for a customer who explicitly asked for tiered buyer groups with custom names (e.g., "Bronze / Silver / Gold").
- If the demo targets D2C — Lauren is a B2B persona; for D2C use the D2C Commerce Buyer Group with Guest browsing.

## Source

`B2B Commerce Demo Builder` doc (Salesforce internal, 2026-05) — Section "Standard Demo User Configuration".
