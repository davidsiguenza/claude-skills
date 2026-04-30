---
name: sf-b2b-demo-builder
description: >-
  Top-level entry point ("package skill") for building a complete Salesforce B2B Commerce demo on an SDO / sandbox / DX org. Routes the conversation between two specialised child skills — `sf-b2b-store-generator` (creates the Experience site, WebStore, pricebooks, buyer groups, branding) and `sf-b2b-catalog-generator` (creates products, categories, variations, pricebook entries, images) — based on a single up-front question: store-only, catalog-only, or both. When both are requested, this skill runs the store first and then hands its IDs / brand / pricebooks straight into the catalog generator so the user is not asked the same questions twice. Use whenever the user says things like "create a B2B Commerce demo", "set up a complete B2B store with products", "I need a new commerce demo", "build me a B2B demo for client X", "do the full B2B Commerce setup", or any request that combines storefront + catalog scope.
---

# Salesforce B2B Commerce Demo Builder — Package Skill

This skill is the **entry point** for building a B2B Commerce demo end-to-end. It does not contain the implementation itself — it just **routes** the conversation between two child skills and **shares context** between them so the user only answers each preflight question once.

## Children

| Child skill | Folder | Owns |
|---|---|---|
| **`sf-b2b-store-generator`** | `.cursor/skills/sf-b2b-store-generator/` | Experience Cloud site (LWR), WebStore, empty `ProductCatalog`, 3 custom pricebooks (List/Sale/VIP), entitlement policy, Standard + VIP buyer groups + accounts + contacts + users, shipping profiles, tax/checkout/payment alignment with the SDO reference store, guest browsing, branding (logo via StaticResource, color tokens on all 4 brandingSets, hero/banner images with CSP Trusted Site, hero copy, navigation), search index. **Stops at "store is publishable but empty".** |
| **`sf-b2b-catalog-generator`** | `.cursor/skills/sf-b2b-catalog-generator/` | Catalog plan tailored to a customer/industry, web research, image sourcing (Wikimedia + vendor domains via `CspTrustedSite`), variation attributes (`ProductAttribute` picklist fields + `ProductAttributeSet` + `ProductAttributeSetItem`), final CSV ready for B2B Commerce Advanced Import (`b2bCatalog/<Customer>/<Customer>.csv`) plus org metadata for variations. **Stops at "CSV + variation metadata ready"; the user runs the Advanced Import in the Setup UI.** |

> Both children also work standalone if invoked directly. This skill is the preferred entry point when the user's intent covers more than one of those scopes — or when they're not sure where to start.

## Phase 0 — The single routing question (mandatory, always run first)

**Hard rule for the agent:** before reading any child SKILL.md, ask the user the question below and **wait for the answer**.

> **What do you want to do for this B2B Commerce demo?**
> 1. **Create the storefront from zero** (Experience site + WebStore + buyer groups + branding, but no products).
> 2. **Generate a product catalog** for an existing storefront (CSV + variations metadata).
> 3. **Both** — create the storefront and then generate the catalog for it.

Echo the choice back ("OK, going with option 3 — full store + catalog for client X") so the user can correct before any work starts.

## Routing logic

### Path A — Storefront only (option 1)

1. Read `.cursor/skills/sf-b2b-store-generator/SKILL.md` and follow it from its **Phase 0 — Preflight questions** all the way through **Phase H — Search index**.
2. **Do NOT** run that skill's Phase I (catalog content). Instead, when the storefront skill ends, finish the conversation by handing back the public URL, buyer credentials and validation plan, plus this one extra line: *"If you decide later you want products, re-invoke me and pick option 2 or 3."*
3. Done.

### Path B — Catalog only (option 2)

The catalog generator needs an org alias, a brand/industry, and one or more `CommerceEntitlementPolicy` names. Before reading its SKILL.md, **scan the workspace** for an existing storefront the user might want to attach the catalog to:

```bash
ls -la branding-work/*/store-ids.json 2>/dev/null
```

- **If one or more `store-ids.json` files exist**, read each one and present a short table to the user:

  | # | Site name | WebStore Id | Org alias | Entitlement policy | Brand source |
  |---|---|---|---|---|---|
  | 1 | Ascendum Commerce Demo | `0ZE...` | `cursorTesting` | `Ascendum Commerce Demo Entitlement Policy` (`1Ce...`) | `branding-work/ascendum-commerce-demo/brand.json` |

  Ask: *"Use one of these as the target store, or build the catalog standalone?"* If the user picks one, jump to **Context handoff schema** below to pre-fill the catalog skill's preflight answers, and only ask the catalog-specific things the storefront does not know (product count, language, currency, tone, price range, extra entitlements).

- **If no `store-ids.json` exists**, fall through to the catalog skill's own preflight (Steps 1, 2b, 2c) so it asks for org / entitlements / company-or-industry from scratch.

Then read `.cursor/skills/sf-b2b-catalog-generator/SKILL.md` and follow it. Done.

### Path C — Storefront + catalog (option 3)

Run the children **sequentially** with context passed between them:

1. **Run `sf-b2b-store-generator` end-to-end** (Phase 0 → Phase H). Do **not** run its Phase I — the catalog skill is more thorough and will replace it.
2. **At the boundary between the two skills**, do the *Phase Bridge* described below — confirm the inferred catalog inputs with the user before continuing.
3. **Run `sf-b2b-catalog-generator`** with the pre-filled context. Skip its Step 1 / 2b questions for the fields the storefront already provided; only ask the catalog-only questions.
4. After the catalog CSV is generated, run the **Catalog import + post-import** steps (see *Phase Bridge — post-import* below) to load the CSV into the WebStore created in step 1.

## Phase Bridge — context handoff (when Path B picks an existing store, or when running Path C)

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

## Phase Bridge — post-import (after the catalog CSV has been generated, only Path B-with-store and Path C)

The `sf-b2b-catalog-generator` skill **stops at "CSV ready"**. To actually load it into the storefront created by `sf-b2b-store-generator`, the orchestrator owns these last steps:

1. **Tell the user how to run the Advanced Import**: Setup → Commerce → the WebStore → *Products* → *Import Products* → upload `b2bCatalog/<Customer>/<Customer>.csv`. (As of 2026-04, this UI step is not reliably automatable via CLI in SDO orgs.)
2. **Wait for the user to confirm import success**, then run the post-import wiring that the storefront skill's Phase I documents:
   - Verify `PricebookEntry` rows landed on all four pricebooks (Standard + List + Sale + VIP) for every SKU.
   - Verify `ProductCategoryProduct` rows link every product to at least the catalog's "All Products" category.
   - Insert `CommerceEntitlementProduct` rows for the entitlement policy from Phase Bridge above (one per product). The Advanced Import does NOT create these.
   - Run `sf commerce search start --store-name '<Site Name>' --targetusername <alias>` to rebuild the search index.
   - Run `sf community publish --name '<Site Name>' --target-org <alias>`.
3. **CMS workspace ↔ site channel link** — the catalog importer puts product images in the org's CMS workspace; that workspace is **not** linked to the new site's channels by default (every Commerce site has its own `Community` + `PublicUnauthenticated` `ManagedContentChannel` rows). Without the link, products show without images on the storefront. The exact PATCH call is in `sf-b2b-store-generator` SKILL → *CMS workspace ↔ site channel: discover and link via CLI*. After linking, the user must publish content from the CMS workspace UI (Connect REST publish currently returns a generic error in SDO).
4. **CMS language gotcha (SDO orgs)** — see `sf-b2b-catalog-generator` SKILL → *CMS language gotcha on SDO orgs*. If the workspace defaults to `sv` (Swedish) and the site renders `en_US`, do the manual **Translations Export → Import** flow. There is no API workaround as of 2026-04.

## Cross-references

- This package skill is the **preferred** entry point for new B2B Commerce demos.
- The two child skills also work standalone — invoke them directly if you only want one scope and you know which one.
- After this skill finishes Path A or Path C, `branding-work/<slug>/store-ids.json` is the source of truth for IDs. Treat it as the per-demo "manifest"; both children read and write to it.
- After this skill finishes Path B or Path C, `b2bCatalog/<Customer>/<Customer>.csv` is the catalog deliverable. Keep it next to `store-ids.json` for handover.

## Maintenance rule

When either child skill changes (new gotcha, new phase, renamed object, new field), update its own SKILL.md — **not** this one. The orchestrator only contains routing logic and the context-handoff schema; everything else lives in the children.
