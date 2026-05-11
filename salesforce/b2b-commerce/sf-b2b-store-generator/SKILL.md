---
name: sf-b2b-store-generator
description: >-
  Bootstraps and aligns a Salesforce B2B Commerce on Core WebStore with Experience Cloud (Commerce Store LWR template) on a Salesforce SDO (Solution Demo Org, last validated 2026-04-23): runs three preflight questions (target org, branding/industry/client, store name), then provisions the Experience site + WebStore, the empty product catalog, the mandatory three-pricebook layout (Standard / Sale / VIP), Standard + VIP buyer groups with accounts and contacts, shipping profiles, tax/checkout alignment with the reference SDO WebStore, guest browsing, Experience Cloud members, buyer Users with permission sets via Apex, and branding — leaving the actual product catalog content as an optional last step (Phase I). Use when creating a B2B commerce storefront in an SDO/sandbox/DX org, repeating the Cursor Commerce Demo setup, or when the user mentions WebStore buyer groups, ShippingConfigurationSet, Guest checkout, or mirroring SDO - B2B Commerce Enhanced.
---

# Salesforce B2B Commerce Store — Repeatable Bootstrap (Cursor Commerce pattern)

Use this skill when provisioning a **new WebStore + Experience site** (template **Commerce Store (LWR)**) and wiring **catalog, pricing, buyers, shipping profiles, tax, and guest browsing**, using **`SDO - B2B Commerce Enhanced`** as the reference store when present.

> **Part of the `sf-b2b-demo-builder` skill package.** If the user asks for "a complete B2B demo" or "store + catalog", invoke `.cursor/skills/sf-b2b-demo-builder/SKILL.md` first — it routes between this skill and `sf-b2b-catalog-generator` and shares context between them so the user is not asked the same questions twice. This skill on its own only builds the **storefront** (no products); when run as part of the package, its Phase I (product catalog content) is replaced by the more thorough `sf-b2b-catalog-generator`.

## Target environment

- **Designed for Salesforce SDO** (Solution Demo Org) — **last validated: 2026-04-23**.
- Assumes the SDO has the **B2B Commerce Enhanced** demo pack pre-installed (`SDO - B2B Commerce Enhanced` WebStore, the Cirrus catalog/pricebooks/entitlement policy, the `B2B Reordering Portal Buyer Profile` profile, and the standard B2B/D2C Commerce permission sets).
- Works on **sandboxes / DX orgs cloned from an SDO**. Will not work on plain scratch orgs without the SDO data pack — many lookup IDs (record types, profiles, permission sets, payment gateway, tax policy) come from the SDO seed.

## Maintenance rule (keep this skill accurate)

- When a setup **phase finishes**, append or tighten the corresponding section below with **commands that worked** and **field names that are valid** for API v66 in this org family.
- When something was **wrong or incomplete**, **replace** that subsection—do not stack contradictory instructions.
- Treat **IDs** as **examples only**; always resolve fresh IDs with SOQL in the target org (`sf data query`).

## Prerequisites

- Salesforce CLI (`sf`), authenticated org alias (e.g. `cursorTesting`).
- Plugin `@salesforce/commerce` (`sf commerce …`) and `@salesforce/plugin-community` (`sf community …`).
- **`sf commerce store create`** requires a **Dev Hub** (`-v`). For sandboxes/non–scratch orgs, use **`sf community create`** + manual alignment (this workflow).

## Phase 0 — Preflight questions (mandatory — always run first)

**Hard rule for the agent:** before touching any phase, ask the user the three questions below and **wait for answers**. Do not assume defaults silently. Capture the answers as session context and propagate them through every later phase (`<alias>`, `<Site Name>`, `<url-path-prefix>`, branding tokens, hero copy).

### Question 1 — Target org

List the orgs the user already has authenticated and ask which one to use, or whether they want to authenticate a new one.

```text
sf org list --json
```

Format the relevant fields (alias, username, instanceUrl, isDefaultUsername, connectedStatus) in a short list and ask the user to pick one. If they want a new org, run **`sf org login web --alias <new-alias>`** and capture the alias.

Once chosen, set the org alias as the default for the rest of the session — every subsequent CLI call uses `-o <alias>`.

### Question 2 — Demo theme / industry / specific client

Ask whether the demo needs:

- **(a) A specific client / brand** — request the **client website URL**. Then scrape it (Defuddle skill or `WebFetch`) to capture: brand name, primary + secondary brand colors (hex), logo URL, hero copy/tagline, product categories, sample product names + descriptions, and any imagery hosts that need to be added to **`CspTrustedSite`** in Phase F2. Save the extracted material as JSON under `branding-work/<slug>/brand.json` so Phase F2 (branding) and the optional Phase I (catalog) can both read from the same source of truth.
- **(b) A specific industry / vertical** without a real client — ask which industry (e.g. *industrial equipment*, *medical supplies*, *fashion B2B wholesale*, *food service*) and any constraints (region, language, target buyer persona). The agent then **invents** brand name + colors + categories + ~10–15 sample SKUs that fit the vertical, and saves them under `branding-work/<slug>/brand.json` with the same shape as option (a).
- **(c) None — generic Cursor Commerce demo** — fall back to the existing Cursor Commerce branding tokens already in the skill (Ascendum reference). Useful when the user just wants to verify the bootstrap works end-to-end.

Always echo the captured answer back ("Branding source: client `ascendum.pt` → primary `#003846`, secondary `#E31D1A`, 5 categories, 14 SKUs detected") so the user can correct before the agent commits to it.

### Question 3 — Store name

Always **propose** a name first, derived from Q2's answer:

- Option (a) client: `<ClientName> Commerce Demo` (e.g. `Ascendum Commerce Demo`).
- Option (b) industry: `<IndustryDescriptor> Commerce Demo` (e.g. `MedSupply Commerce Demo`).
- Option (c) generic: `Cursor Commerce Demo`.

Ask the user to **confirm or override**. From the chosen name derive:

- `<Site Name>` (full name, used for `sf community create --name`, `WebStore.Name`, branding strings).
- `<url-path-prefix>` — lowercased, only `[a-z0-9]`, max 30 chars (e.g. `Ascendum Commerce Demo` → `ascendumcommercedemo`). Salesforce will append `vforcesite` to the public URL — that's expected.
- `<site-slug>` for local working folders (`branding-work/<slug>/`, `catalog-work/<slug>/`).

### Question 4 — Storefront currency

Ask **which ISO 4217 currency code** the storefront should price in (e.g. `USD`, `EUR`, `GBP`, `CAD`, `AUD`, `BRL`, `MXN`). Default to `USD` only if the user has no preference — do **not** silently assume USD based on the org's corporate currency. Store it as `<CURRENCY>`.

**Critical reason to ask up-front**: the currency is stamped on every `Pricebook2`, `PricebookEntry`, and User record the skill creates. Getting it wrong means the storefront silently renders **"Price Unavailable"** for every product even though pricebooks, buyer groups, and entries all look correct (the User's `DefaultCurrencyIsoCode` must match the `PricebookEntry.CurrencyIsoCode` for the pricing engine to resolve). Retrofitting after the fact requires updating pricebooks, users (including the site Guest User), entries, and often republishing the site.

Before proceeding, verify the org has the currency enabled:

```bash
sf data query -q "SELECT IsoCode, IsActive, IsCorporate FROM CurrencyType WHERE IsoCode = '<CURRENCY>'" -o <alias>
```

If the query returns 0 rows, **try the CLI/API path before falling back to Setup**. In SDOs and many sandboxes, multi-currency can be enabled and currencies can be added from the CLI; do not tell the user it is impossible until the target org rejects these commands.

```bash
# 1. Detect whether the org already exposes CurrencyType.
sf data query -q "SELECT IsoCode, IsActive, IsCorporate FROM CurrencyType ORDER BY IsoCode" -o <alias>

# 2. If CurrencyType is not supported yet, try to enable multi-currency through OrgPreferenceSettings.
# Some org families expose this setting; some return an error and require the UI fallback below.
sf data update record -s OrgPreferenceSettings -i OrgPreferenceSettings -v "IsMultiCurrencyEnabled=true" -o <alias>

# 3. Add or activate the target currency.
sf data create record -s CurrencyType -v "IsoCode=<CURRENCY> ConversionRate=1.0 DecimalPlaces=2 IsActive=true" -o <alias>
# If the create reports a duplicate/inactive row, update the existing CurrencyType instead:
sf data update record -s CurrencyType -i <CURRENCY_TYPE_ID> -v "IsActive=true" -o <alias>

# 4. Re-query and proceed only when the row is active.
sf data query -q "SELECT Id, IsoCode, IsActive, IsCorporate, ConversionRate FROM CurrencyType WHERE IsoCode = '<CURRENCY>'" -o <alias>
```

Use a realistic `ConversionRate` if the demo currency differs from the corporate currency and pricing comparisons matter; for isolated demos, `1.0` is acceptable as a bootstrap default and can be adjusted later. If the org rejects `OrgPreferenceSettings` or `CurrencyType` DML, then use the UI fallback: Setup → Company Information → **Activate Multiple Currencies**, then Setup → Manage Currencies → **New** / **Activate** `<CURRENCY>`.

If `IsCorporate = true` and matches `<CURRENCY>`, no further action is needed. If the corporate currency is different, the skill still works but **every** buyer-facing User must be explicitly set to `<CURRENCY>` (see Phase E and Phase A's Guest User setup), and every pricebook / pricebook entry created by this skill must carry the same `CurrencyIsoCode`.

Persist `<CURRENCY>` to `branding-work/<slug>/brand.json` → `currency` so the catalog generator (Phase I, or the `sf-b2b-catalog-generator` child skill) uses the same value without re-asking.

> **Do not start Phase A** until all four answers are captured and confirmed by the user.

### Execution order after preflight

Run phases in **this** order. Phase I (catalog content) is **optional and deferred** — only run it if the user explicitly asks for it after the rest of the store is up.

1. Phase A — Experience site + WebStore
2. Phase B — Empty catalog + 3-pricebook scaffolding (no products yet)
3. Phase C — WebStore checkout / tax / payment alignment
4. Phase D — Shipping profiles
5. Phase E — Buyer groups + accounts + contacts (Standard + VIP)
6. Phase F — Login + permission sets (full Phase F.1 → F.4)
7. Phase F2 — Branding (uses `branding-work/<slug>/brand.json` from Q2)
8. Phase G — Org-level gateway sanity check
9. Phase H — Search index (empty index is fine — store is technically sellable as soon as products are added)
10. **Phase Z — Self-validation checklist (mandatory final gate before handover)** — the agent runs every SOQL/REST check below itself and prints a pass/fail table. The store is **only** considered ready when every check is green. If any check fails, the agent goes back to the corresponding phase, fixes it, and re-runs Phase Z.
11. **Phase I (optional, deferred) — Product catalog content** (categories, products, `PricebookEntry` rows, `ProductCategoryProduct`, `CommerceEntitlementProduct`, images). Hand the storefront back to the user and ask if they want to proceed with Phase I or manage the catalog separately.

## Reference resources — official B2B Commerce open-source components

Salesforce publishes the **source code for every stock B2B Commerce LWC** (cart, checkout, productCard, siteLogo, navigation, etc.) under MIT license. **Always check this repo before guessing how a storefront component reads its data, or before building a custom replacement.** It is the source of truth when:

- A stock LWC silently swallows your config (e.g. it reads from `ProductMedia.ElectronicMediaId` instead of `Product2.DisplayUrl` — exactly why our parked image issue exists).
- You need to **fork** a component to override its template or styles in shadow DOM (e.g. the parked logo issue with `dxp_content_layout:siteLogo`).
- You want to subclass / wrap an existing component to add behavior (custom `productCard`, custom `cartItem` price formatter, etc.).
- You're diagnosing why CSS overrides in `sfdc_cms__styles/styles.css` don't reach inner elements — read the component's `.html` template, see its `<template lwc:render-mode="shadow">` declaration, then decide whether to fork it or use a CSS Custom Property exposed by the component.

**Repo:** [forcedotcom/b2b-commerce-open-source-components](https://github.com/forcedotcom/b2b-commerce-open-source-components/tree/main/force-app/main/default/sfdc_cms__lwc) — all components live under `force-app/main/default/sfdc_cms__lwc/<componentName>/`.

**Components specifically relevant to flows in this skill:**

| Stock component | Where it lives in this skill | Why you might fork it |
|---|---|---|
| `siteLogo` (referenced as `dxp_content_layout:siteLogo`) | Phase F2 — Branding (logo parked, §11.2 of `USER_MANUAL_STEPS.md`) | LWC renders the logo `<img>` in **shadow DOM**, so root `styles.css` cannot retarget it. Fork to swap the image src or wrap with custom slot. |
| `productCard` family (`searchProductCard`, `productCardContent*`) | Phase I — image upload parked (§11.4) | Reads images from `ProductMedia → ManagedContent`. Fork if you want it to fall back to `Product2.DisplayUrl` or to an external CDN URL. |
| `commonFormattedPrice` / `commonFormattedCurrency` | Phase F2 + Phase G.1 (Salesforce Pricing engine) | The "Price Unavailable" string lives in this component's labels. Fork to change the empty-state text or inject a custom currency policy. |
| `commonBreadcrumbs`, `commonDrilldownNavigation*` | Phase I — `All Products` PLP entry | Useful when the buyer wants the catalog tree to render as a side filter rather than a top nav. |
| `cartItem`, `cartContents`, `cartSummary` | Future checkout customization | Fork these to add per-row business rules (PO number per line, quantity unit conversions, etc.). |
| `checkoutDeliverymethod*`, `checkoutInputAddress`, `checkoutBillingInfo` | Phase D (shipping) + future checkout work | Fork to inject custom country pickers, EU VAT collection, B2B-specific checkout fields. |

**How to use a forked component in this org:**

1. Clone (or sparse-clone) only the component subfolder you need from the GitHub repo.
2. Copy it into your local SFDX project under `force-app/main/default/lwc/<myCustomName>/` — **rename** it (e.g. `cursorProductCard`) so it doesn't collide with the stock one in the **`sfdc_cms__lwc`** namespace.
3. Edit the `.js` / `.html` / `.css` as needed.
4. Deploy with `sf project deploy start --target-org <alias> --source-dir force-app/main/default/lwc/<myCustomName>`.
5. In Experience Builder (or by editing the `sfdc_cms__view` JSON in the bundle), **replace** the stock component reference with `c-<my-custom-name>` (kebab-case).
6. Re-publish the site (`sf community publish ...`) and hard-refresh.

**Caveat:** The repo tracks the **current** Salesforce release. If your org is on an older release, some component dependencies (utility modules under `commonConfig`, `commerceErrors`, etc.) may not match what's on the platform. Pin the repo to the tag matching your org's API version when forking for production.



## Phase A — Experience site + WebStore (Commerce Store LWR)

1. List templates: `sf community list template --target-org <alias> --json` — confirm **`Commerce Store (LWR)`** exists.
2. Create site (async job):  
   `sf community create --name '<Site Name>' --template-name 'Commerce Store (LWR)' --url-path-prefix <valid-prefix> --target-org <alias> --json`  
   Poll **`BackgroundOperation`** until **Complete**.
3. Retrieve **`Network`** (site) and **`WebStore`** created for that site (`Name` usually matches site name). Note **`WebStore.Id`**, **`Network.Id`**, **`UrlPathPrefix`** (Salesforce may append `vforcesite`).
4. Publish: `sf community publish --name '<Site Name>' --target-org <alias> --json`; poll **`BackgroundOperation`** if needed.
5. If the site stays in **Preview / Under Construction** for the public URL, set **`Network.Status = Live`** and re-run publish as needed:

```text
sf data update record -s Network -i <NETWORK_ID> -o <alias> -v "Status=Live OptionsSelfRegistrationEnabled=true"
sf community publish --name '<Site Name>' --target-org <alias> --json
```

**Self-registration:** set `OptionsSelfRegistrationEnabled=true`. If self-registration still fails or users are created without catalog access, set **`Network.SelfRegProfileId`** explicitly to the **community buyer** profile used for B2B storefront users (typically **`B2B Reordering Portal Buyer Profile`**—resolve **`Profile.Id`** with SOQL in the target org). The SDO reference site may show **`SelfRegProfileId` null in SOQL** while the storefront works—Commerce/LWR sites cloned via **`sf community create`** often need **`SelfRegProfileId`** set for reliable signup.

> **CRITICAL — Self-registration buyer profile must be `Customer Community Plus` (CCP).** The dropdown for `SelfRegProfileId` only shows profiles that are (a) already Site Members and (b) based on a `Customer Community Plus` or `Customer Community Plus Login` license. Any other license (`External Apps`, `Partner Community`, `Salesforce`) causes either the dropdown to filter out the profile or the self-reg to fail with `FIELD_INTEGRITY_EXCEPTION` during PSG/PermSet assignment. The **only safe way** to create the shopper profile is to **clone** an existing CCP profile (e.g. `B2B Reordering Portal Buyer Profile`) via Setup → Profiles → Clone — Salesforce fixes the license to CCP during the clone and **it cannot be changed afterward**. Verify: `SELECT UserLicense.Name, UserType FROM Profile WHERE Name = '<Your Shopper Profile>'` — must return `UserLicense.Name=Customer Community Plus` and `UserType=PowerCustomerSuccess`. If it returns `Salesforce` or `External Apps`, delete the profile and re-clone.

> **CRITICAL — Permission Sets assigned on self-reg must not declare `objectPermissions` on `BuyerGroup`/`BuyerGroupMember`.** If the PermissionSet (or PSG) assigned at self-reg time contains `<viewAllRecords>true</viewAllRecords>` or any `<allowRead>` / `<objectPermissions>` block for `BuyerGroup` or `BuyerGroupMember`, the self-reg transaction fails with `FIELD_INTEGRITY_EXCEPTION: The user license doesn't allow the permission: View All BuyerGroup` (or `Read BuyerGroup`). These objects' access is gated by the `B2B Buyer` PSL, which is assigned **after** the self-reg (step 6 below). Remove all `BuyerGroup`/`BuyerGroupMember` object permissions from every PermissionSet or PSG used in self-registration. Also remove `<license>` from the permset XML — a license-locked permset cannot be assigned to a CCP user.

> **CRITICAL — Standard Salesforce self-reg does NOT complete B2B buyer onboarding.** It creates `User + Contact + PersonAccount` and optionally assigns a `Profile` and `PSG`, but it does **not** perform the 4 steps required for B2B catalog access. After self-reg you must run a post-registration script (Apex Queueable or anonymous Apex) that:
> 1. Activates `BuyerAccount` (`BuyerStatus=Active`, then update `IsActive=true` in a separate DML — same two-step pattern as Phase E's bootstrap accounts).
> 2. Adds the user's Account as `BuyerGroupMember` to at least one `BuyerGroup` linked to the `WebStore`.
> 3. Assigns `PermissionSetLicenseAssignment` for `B2B Buyer` PSL.
> 4. Assigns `PermissionSetAssignment` for `B2BBuyer` permset.
> Without these 4 steps the logged-in shopper receives `INSUFFICIENT_ACCESS, You don't have access to stores.` even though the self-reg appeared to succeed.
>
> Use a Queueable Apex triggered on `User AfterInsert/AfterUpdate` (filter by shopper ProfileId) to automate this — plain trigger or synchronous Flow cannot do `PermissionSetLicenseAssignment` DML in the same transaction as portal-user creation (MIXED_DML_OPERATION error).
>
> **Session cache caveat:** if the buyer logged in before steps 1–4 were applied, the browser has cached the "no access" state. Closing all windows and reopening in incognito (fresh session) is required — simply logging out/in in the same browser is not enough.

**Embedded login (guest browse vs login redirect):** align **`Network`** with the working reference store (`SDO - B2B Commerce Enhanced`) — in particular **`OptionsEmbeddedLoginEnabled=true`**. If this is **`false`**, anonymous visitors may be redirected to login instead of browsing the catalog as guests even when **`WebStore.OptionsGuestBrowsingEnabled=true`**.

> **CRITICAL — `SelfRegProfileId` order:** you cannot set `Network.SelfRegProfileId` in the same call as `Status=Live` (or any other update) until the buyer profile is **already a member of the site**. Otherwise the update fails with:
> *"You can only select profiles that are associated with the experience."*
>
> Run the three calls below **in this exact order** — never collapse them into one update:

```text
# 1. Activate the network and turn on login features WITHOUT SelfRegProfileId
sf data update record -s Network -i <NETWORK_ID> -o <alias> -v "Status=Live OptionsSelfRegistrationEnabled=true OptionsAllowInternalUserLogin=true OptionsEmbeddedLoginEnabled=true OptionsGuestChatterEnabled=true"

# 2. Add the buyer profile (and the Commerce permission sets) to the site as members
sf data create record -s NetworkMemberGroup -o <alias> -v "NetworkId=<NETWORK_ID> ParentId=<B2B_BUYER_COMMUNITY_PROFILE_ID>"
# (also do the same for B2B_Commerce_Guest_Browser_Access, B2BCommerce_Community_Access, D2C_Commerce_Guest_User_Access — see Phase F.1)

# 3. NOW set SelfRegProfileId (separate call, after the profile is a member)
sf data update record -s Network -i <NETWORK_ID> -o <alias> -v "SelfRegProfileId=<B2B_BUYER_COMMUNITY_PROFILE_ID>"
```

**Public access + guest-safe navigation (CLI — MANDATORY, never deferred to UI):** every new store **must** end Phase A with guest browsing fully wired so anonymous visitors can reach the catalog menu and PLP. Skipping this is the #1 way to ship a "broken" demo where guests are bounced to login. Three settings work as a unit and **all** must be set in this phase:

1. **`WebStore.OptionsGuestBrowsingEnabled = true`** — turns the WebStore itself into a guest-capable store. Update at the WebStore level; this is the only setting reachable purely via Data API:

```text
sf data update record -s WebStore -i <WEBSTORE_ID> -o <alias> -v "OptionsGuestBrowsingEnabled=true OptionsGuestCartEnabled=true OptionsGuestCheckoutEnabled=true OptionsPreserveGuestCartEnabled=true"
```

2. **`sfdc_cms__site/<SITE>/content.json` → `contentBody.authenticationType = AUTHENTICATED_WITH_PUBLIC_ACCESS_ENABLED`** — controls whether the LWR runtime even allows anonymous requests. New Commerce Store sites default to **`AUTHENTICATED`** (login required) which silently breaks guest browsing even when `OptionsGuestBrowsingEnabled=true`. Must be patched in the `DigitalExperienceBundle` and deployed.

3. **`NavigationMenu` → Storefront Categories → `publiclyAvailable = true`** — controls whether the mega-menu surfaces categories to guests. The Builder checkbox is just a wrapper around this metadata flag. Without it, guests load the home page but see an empty header menu.

The **Storefront Categories → Publicly available** flag maps to **`NavigationMenu`** metadata for the network's default navigation (name pattern commonly **`SFDC_Default_Navigation_<SanitizedSiteName>`**) → **`navigationMenuItem`** where **`target`** = **`StorefrontCategories`** → **`publiclyAvailable`** = **`true`**.

Discover the menu API name:

```text
sf org list metadata --metadata-type NavigationMenu --target-org <alias>
```

Retrieve, edit, deploy, publish:

```text
sf project retrieve start --target-org <alias> --metadata "DigitalExperienceBundle:site/<SITE_BUNDLE_NAME>" --output-dir ./retrieve-site
sf project retrieve start --target-org <alias> --metadata "NavigationMenu:<DEFAULT_NAV_API_NAME>" --output-dir ./retrieve-nav
```

- In **`digitalExperiences/site/<SITE_BUNDLE_NAME>/sfdc_cms__site/<siteFolder>/content.json`**, set `"authenticationType" : "AUTHENTICATED_WITH_PUBLIC_ACCESS_ENABLED"` inside **`contentBody`** (align with **`SDO_B2B_Commerce_Enhanced1`** bundle).
- In **`navigationMenus/<NavName>.navigationMenu-meta.xml`**, set **`<publiclyAvailable>true</publiclyAvailable>`** on the **Storefront Categories** item.

Then:

```text
sf project deploy start --source-dir force-app --target-org <alias>
sf community publish --name '<Site Name>' --target-org <alias> --json
```

Verify after deploy + publish (all three must hold):

```text
# 1. WebStore guest flag
sf data query -q "SELECT OptionsGuestBrowsingEnabled, OptionsGuestCartEnabled, OptionsGuestCheckoutEnabled FROM WebStore WHERE Id = '<WEBSTORE_ID>'" -o <alias>

# 2. Site authenticationType (read JSON back from the deployed bundle)
grep -A1 '"authenticationType"' force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/sfdc_cms__site/<SITE_BUNDLE>/content.json
# Expected: "authenticationType" : "AUTHENTICATED_WITH_PUBLIC_ACCESS_ENABLED"

# 3. NavigationMenu Storefront Categories
grep -B2 -A4 '<target>StorefrontCategories</target>' force-app/main/default/navigationMenus/*.navigationMenu-meta.xml
# Expected: <publiclyAvailable>true</publiclyAvailable> on the StorefrontCategories item
```

This workspace includes a worked example package after retrieve/patch (same org family): **`sf-commerce-guest-access-fix/`** — deploy only when targeting the matching **`DigitalExperienceBundle`** name (**`site/Cursor_Commerce_Demo1`**).

**Experience Builder (fallback only if bundle deploy is blocked):** same outcome as editing **Settings → General** (public access) and **Navigation → Storefront Categories → Publicly available**, then **Publish**. Treat this strictly as a recovery path — the CLI/metadata route is the contract for this skill, so the agent never reaches the user-handover stage by relying on UI checkboxes.

**Default buyer group on self‑registration (mirrors SDO *Stores → Buyer access*):** use **`CommerceConfigRelatedRecord`** with **`ContextId` = the Experience `Network` (site) Id**—**not** the `WebStore` Id. SDO B2B Enhanced uses **`ConfigUseCase=SelfRegistration`**, **`ConfigKey=DefaultBuyerGroup`**, and **`ConfigReferenceId=<BuyerGroup.Id>`** (Account-based group for new B2B accounts) plus a second row **`ConfigKey=AccountRecordType`** for the B2B account record type.

```text
sf data create record -s CommerceConfigRelatedRecord -o <alias> -v "ContextId=<NETWORK_ID> ConfigUseCase=SelfRegistration ConfigKey=DefaultBuyerGroup ConfigReferenceId=<BUYER_GROUP_ID>"
sf data create record -s CommerceConfigRelatedRecord -o <alias> -v "ContextId=<NETWORK_ID> ConfigUseCase=SelfRegistration ConfigKey=AccountRecordType ConfigReferenceId=<RECORD_TYPE_ID>"
```

**Org setting *Buyer group extensibility*:** `Setup → Commerce → Settings` maps to metadata **`Commerce.settings` → `buyerGroupExtensibility`**. Set to **`true`** (metadata deploy) to align with the “Buyer group extension / advanced buyer access” options in the Commerce app.

**Guest catalog / search (LWR):** the **template’s** `WebStore.GuestBuyerProfileId` often points to a **store-specific** `GuestBuyerProfile` that is **not** equivalent to the org’s **B2B Guest Buyer Profile** used by SDO. The field is often **not updateable** via Data API; in **Commerce → Store** (or **Experience** store settings), set the **Guest buyer profile** to the same record as the reference store (e.g. **B2B Guest Buyer Profile**, `3K0g...` in many SDO orgs) if guests cannot see the catalog menu or product search. This is separate from `Profile` (community user) FLS but commonly the missing piece when `Role=Market` + `BuyerGroupBuyerCriteria` is already correct.

## Phase B — Empty catalog + 3-pricebook scaffolding (mandatory base)

Phase B sets up the **container objects** the rest of the store needs (catalog, entitlement policy, three pricebooks) **without** any products or `PricebookEntry` rows. The store is fully wired and publishable — buyers can log in, see the empty PLP, and the pricing engine has every reference it needs the moment products are added in **Phase I (optional, deferred)**.

### Step 1 — Empty `ProductCatalog` + `CommerceEntitlementPolicy` + mandatory "All Products" category

```text
sf data create record -s ProductCatalog            -v "Name='<Site Name> Catalog'           Description='<Site Name> empty catalog scaffolding (products added in Phase I).'" -o <alias>
sf data create record -s CommerceEntitlementPolicy -v "Name='<Site Name> Entitlement Policy' CanViewPrice=true CanViewProduct=true IsActive=true"                                  -o <alias>
```

Capture `<CATALOG_ID>` and `<POLICY_ID>` for later phases.

**Mandatory "All Products" category — create immediately, before Phase E:**

```text
sf data create record -s ProductCategory -v "Name='All Products' CatalogId=<CATALOG_ID> Description='Browse the full catalog.' SortOrder=1" -o <alias>
```

Capture `<ALL_PRODUCTS_CAT_ID>` and persist it to `branding-work/<slug>/store-ids.json` so the catalog generator (or anyone running Phase I) can attach every imported product to it. **Every** product that lands in the store **must** end up with a `ProductCategoryProduct` row pointing at this category in addition to its specific category — that's how the storefront mega-menu surfaces an "All" entry that never goes empty even if a category is mid-edit. Skipping this step is the parked failure mode where a fresh store's mega-menu has zero entries until at least one specific category exists.

### Step 2 — Wire the catalog to the WebStore

```text
sf data create record -s WebStoreCatalog -v "SalesStoreId=<WEBSTORE_ID> ProductCatalogId=<CATALOG_ID>" -o <alias>
```

> `WebStoreCatalog.ProductCatalogId` is **non-updateable**. If you ever need to swap the catalog later, delete the row and create a new one.

### Step 3 — Default 3-pricebook layout (mandatory for every new store)

Every new WebStore created with this skill **must** ship with **three active custom pricebooks** wired through `WebStorePricebook`. This guarantees we always have a "list / sale / VIP" tier and a clean mapping for the Standard and VIP buyer groups created in Phase E. The pricebooks are created **empty**; `PricebookEntry` rows are added per SKU during Phase I (optional, deferred).

> **CRITICAL — DO NOT use the org `Standard Price Book` (`IsStandard=true`) as the WebStore "list" pricebook.** Cursor / SDO orgs reject it on `WebStorePricebook` insert with:
> *"Standard Price Book isn't available to stores or buyer groups. Try another price book."*
> The same restriction applies to `BuyerGroupPricebook`. The org Standard Price Book stays as the source-of-truth for `PricebookEntry` rows behind the scenes (Salesforce requires it for any `PricebookEntry`), but the WebStore must reference its own **custom** "List" pricebook. The reference SDO B2B Commerce Enhanced store follows the same pattern (it uses `Cirrus Price Book` + `Cirrus Silver Price Book`, never the org Standard).

| Pricebook | Role | `Pricebook2.Name` (default) | Active? | Notes |
|---|---|---|---|---|
| **List** (custom — replaces org Standard for WebStore wiring) | List price for the storefront | **`<Site Name> List Price Book`** | yes | Acts as the strikethrough/source-of-truth list price for buyers. In Phase I, every SKU gets a `PricebookEntry` here matching the same `UnitPrice` as the org Standard Price Book entry. |
| Sale | Discounted list for the **Standard Customers Buyer Group** | **`<Site Name> Sale Price Book`** | yes | Same SKUs as List, lower price. Linked to the **Standard** buyer group with `Priority=1`. The List pricebook is linked to the same group with `Priority=2` so it renders as strikethrough. |
| VIP | Deepest discount for the **VIP Customers Buyer Group** | **`<Site Name> VIP Price Book`** | yes | Same SKUs as List/Sale, the lowest price. Linked to the **VIP** buyer group with `Priority=1`; List added at `Priority=2` for strikethrough. |

Create all three custom pricebooks and wire them to the WebStore as part of bootstrap. **Stamp `<CURRENCY>` from Phase 0 Q4 on every Pricebook2 and every subsequent `PricebookEntry`** — this is what ties pricing to the currency the user chose. In multi-currency orgs the field is `CurrencyIsoCode`; if the org is single-currency this field does not exist and should be omitted.

```text
# Multi-currency org (has CurrencyType table)
sf data create record -s Pricebook2 -v "Name='<Site Name> List Price Book' IsActive=true CurrencyIsoCode=<CURRENCY> Description='List/strikethrough pricebook (org Standard PB cannot be wired to WebStore in this org family).'" -o <alias>
sf data create record -s Pricebook2 -v "Name='<Site Name> Sale Price Book' IsActive=true CurrencyIsoCode=<CURRENCY>" -o <alias>
sf data create record -s Pricebook2 -v "Name='<Site Name> VIP Price Book'  IsActive=true CurrencyIsoCode=<CURRENCY>" -o <alias>

sf data create record -s WebStorePricebook -v "WebStoreId=<WEBSTORE_ID> Pricebook2Id=<LIST_PB_ID> IsActive=true" -o <alias>
sf data create record -s WebStorePricebook -v "WebStoreId=<WEBSTORE_ID> Pricebook2Id=<SALE_PB_ID> IsActive=true" -o <alias>
sf data create record -s WebStorePricebook -v "WebStoreId=<WEBSTORE_ID> Pricebook2Id=<VIP_PB_ID>  IsActive=true" -o <alias>
```

Detect multi-currency with `sf data query -q "SELECT IsoCode FROM CurrencyType LIMIT 1" -o <alias>` — if the query errors with `sObject type 'CurrencyType' is not supported`, the org is single-currency, drop the `CurrencyIsoCode=` from each command. Otherwise always include it.

Set the strikethrough on the WebStore so the storefront renders both prices once products exist:

```text
sf data update record -s WebStore -i <WEBSTORE_ID> -v "StrikethroughPricebookId=<LIST_PB_ID>" -o <alias>
```

Save the resolved IDs (`<CATALOG_ID>`, `<POLICY_ID>`, `<LIST_PB_ID>`, `<SALE_PB_ID>`, `<VIP_PB_ID>`) to `branding-work/<slug>/store-ids.json` so Phase E and Phase I can read them without re-querying. The org `Standard Price Book` ID is still needed in Phase I (to insert one `PricebookEntry` per SKU there as the platform's source-of-truth requirement), but it never appears in `WebStorePricebook` or `BuyerGroupPricebook`.

### Critical sObject quirks (apply throughout the catalog flow)

- **`WebStoreCatalog.ProductCatalogId` is non-updateable.** To switch a store's catalog you must **delete the existing `WebStoreCatalog` row and create a new one**.
- **`CommerceEntitlementBuyerGroup` is non-updateable** ("entity type cannot be updated: Entitlement Buyer Group"). To swap a buyer group's policy, **delete and recreate** the link row.
- **`ProductCatalog`** has **no** `CatalogCode` field. Just `Name` + `Description`.
- **`BuyerGroup`** has **no** `BuyerGroupPriority` field — pricebook precedence is on **`BuyerGroupPricebook.Priority`** (lowest number wins).
- **Nested semi-join sub-selects are not supported.** Resolve buyer-group / entitlement chains in two steps: first query parent IDs, then `IN` against them.

> **No products yet.** Phase B intentionally stops here. Products, categories, `PricebookEntry` rows and product images are handled in **Phase I (optional, deferred)** at the end of this skill. The store is fully publishable and buyers can log in to an empty PLP.

## Phase C — Align WebStore checkout/tax/pricing/guest behavior with SDO B2B Enhanced

Compare **`WebStore`** records:

```text
sf data get record -s WebStore -i <SDO_WEBSTORE_ID> -o <alias> --json
sf data get record -s WebStore -i <YOUR_WEBSTORE_ID> -o <alias> --json
```

Copy **scalar flags** from SDO → target (examples that mattered):

- **Tax:** `DefaultTaxPolicyId`, `DefaultTaxLocaleType` (e.g. **Net**).
- **Location:** `LocationId` (SDO uses the location tied to **SDO - B2B Commerce**).
- **Price books on store:** `SortByPricebookId`, `StrikethroughPricebookId`.
- **Checkout:** `CheckoutTimeToLive`, `GuestCartTimeToLive`, `OrderActivationStatus`, `OrderLifeCycleType`.
- **Pricing strategy:** `PricingStrategy`, `ProductGrouping`.
- **Guest commerce:** `OptionsGuestBrowsingEnabled`, `OptionsGuestCartEnabled`, `OptionsGuestCheckoutEnabled`, `OptionsPreserveGuestCartEnabled`, plus cart/async flags as per SDO.

Update:

```text
sf data update record -s WebStore -i <YOUR_WEBSTORE_ID> -o <alias> -v "Field=value ..."
```

**Known limits:**

- **`GuestBuyerProfileId`** may be **not writeable** via CLI/Apex for the running admin (field-level security). The template still assigns a guest profile; verify **`GuestBuyerProfileId`** in UI if parity with SDO’s profile ID is mandatory.

### StoreIntegratedService — payments, Salesforce Tax, and other integrations

Template-created stores often ship **without** **`StoreIntegratedService`** rows while SDO/reference stores already have them. Without these, **Commerce checkout setup** may show **no payment provider**, even if **`PaymentGateway`** (**SalesforcePG**) exists org-wide.

**Discover the reference store’s integrations:**

```text
sf data query -q "SELECT Integration, ServiceProviderType FROM StoreIntegratedService WHERE StoreId = '<SDO_WEBSTORE_ID>' ORDER BY ServiceProviderType" -o <alias>
```

Typical **`SDO - B2B Commerce Enhanced`** mappings (`ServiceProviderType` → `Integration`):

| ServiceProviderType | Integration value |
|---------------------|-------------------|
| Shipment | `Shipment__B2B_STOREFRONT__StandardShipment` |
| Tax | `Tax__COMPUTE_TAXES` (**Salesforce Tax / compute taxes**) |
| Price | `Price__B2B_STOREFRONT__StandardPricing` |
| Promotions | `Promotions__B2B_STOREFRONT__StandardPromotions` |
| Payment | `<PaymentGateway.Id>` — resolve **`SELECT Id FROM PaymentGateway WHERE PaymentGatewayName = 'SalesforcePG' AND Status = 'Active'`** |
| Inventory | `Inventory__CHECK_INVENTORY` |

**Create rows for your `WebStore.Id`** (one row per line):

```text
sf data create record -s StoreIntegratedService -o <alias> -v "StoreId=<YOUR_WEBSTORE_ID> Integration=<Integration> ServiceProviderType=<Shipment|Tax|Price|Promotions|Payment|Inventory>"
```

**Tax UI alignment:** Keep **`WebStore.DefaultTaxLocaleType = Net`** and **`WebStore.DefaultTaxPolicyId`** = the **`TaxPolicy`** named **Default Tax Policy** (verify with `SELECT Id, Name FROM TaxPolicy`). **`Tax__COMPUTE_TAXES`** plus those **`WebStore`** fields correspond to **Salesforce Tax Service** with **Net** and the default policy in the admin UI.

## Phase D — Shipping profiles (ShippingConfigurationSet) — required for shipping at checkout

**Observation:** If **`ShippingConfigurationSet`** rows are **missing** for the new **`WebStore`**, checkout may not offer the same shipping behavior as SDO even when **`LocationId`** matches.

### D.1 — API-creatable objects (do these first)

1. List SDO’s sets:

```text
sf data query -q "SELECT Id, Name, TargetRecordId, IsDefault, ProcessTime, ProcessTimeUnit FROM ShippingConfigurationSet WHERE TargetRecordId = '<SDO_WEBSTORE_ID>'" -o <alias>
```

2. For each row, **create** the same structure for **your** `WebStore.Id`:

```text
sf data create record -s ShippingConfigurationSet -o <alias> -v "Name='<Name>' TargetRecordId=<YOUR_WEBSTORE_ID> IsDefault=<true|false> ProcessTime=<n> ProcessTimeUnit=<Days|Hours|...>"
```

3. **`ShippingConfigSetProduct`**: For **Special Handling** (or equivalent) profile, clone **`Product2Id`** rows from SDO’s matching **`ShippingProfileId`**:

```text
sf data query -q "SELECT Product2Id FROM ShippingConfigSetProduct WHERE ShippingProfileId = '<SDO_SHIPPING_PROFILE_ID>'" -o <alias>
sf data create record -s ShippingConfigSetProduct -o <alias> -v "ShippingProfileId=<NEW_PROFILE_ID> Product2Id=<PRODUCT2_ID>"
```

**Platform rule:** A given **`Product2Id`** may only appear **once per WebStore** across shipping profiles—do not duplicate across two profiles for the same store (error: product already exists in a shipping profile for the web store).

### D.2 — Seed RateGroup + RateArea via API (otherwise the UI hides the "New Rate" button)

**Critical UX gotcha:** The Commerce Setup UI **does not let you create the first `ShippingRateGroup` from scratch** on a profile that doesn't have one. If a `ShippingConfigurationSet` has no `ShippingRateGroup`, the profile renders with no "Shipping Zones" section at all — only "Shipping Profile Details" and "Delete Shipping Profile". You cannot click "New Zone" because the container isn't there.

**Workaround:** seed one `ShippingRateGroup` + **at least two `ShippingRateArea` zones** (United States + Europe) per profile via API, then the UI renders the zones section and lets you add rates.

**Mandatory zones (default for every demo):**
- **United States** — `Countries='US'`
- **Europe** — `Countries='ES, FR, PT, IT, DE'` (extend with the demo's target country if it's elsewhere in Europe)

**If the demo targets a country outside US/Europe** (e.g. Mexico, Brazil, Japan), add an extra `ShippingRateArea` for that region in addition to (or replacing) the defaults. Always confirm the country list with the user before seeding.

```text
# For each ShippingConfigurationSet created in D.1:
RGID=$(sf data create record -s ShippingRateGroup -v "Name='Shipping Rate Group' ShippingProfileId=<PROFILE_ID>" -o <alias> --json | python3 -c "import json,sys; print(json.load(sys.stdin)['result']['id'])")

# United States zone (always)
sf data create record -s ShippingRateArea -v "Name='United States' Countries='US' ShippingRateGroupId=$RGID" -o <alias>

# Europe zone (always — extend countries if demo targets a specific EU country)
sf data create record -s ShippingRateArea -v "Name='Europe' Countries='ES, FR, PT, IT, DE' ShippingRateGroupId=$RGID" -o <alias>

# Add extra zones here if the demo targets a country outside US/Europe
```

### D.3 — MANDATORY manual UI step (rates)

**Critical finding (2026-05-05, Dentaid demo):** `StandardShippingRate.ShippingCarrierMethodId` is **read-only at metadata level** (both REST API and Apex fail with `Field is not editable`). Creating rates via API leaves that field `null`, so `StandardShipment` returns zero methods at checkout → error 400 `can't deliver this item` or `Region Code is not supported`.

**The only path that works** is the Commerce Setup UI, which creates the `ShippingCarrierMethod` automatically when you add a rate with a new Rate Name:

1. **Setup → Commerce → Stores → `<Your Store>` → Administration → Checkout → Shipping** (NOT "Shipping Rates" in classic Setup — that one is for Order Management, not checkout).
2. Open each profile seeded in D.2 — at minimum the **default** one (`IsDefault=true`).
3. If you need more zones than the ones seeded in D.2, click **New Zone** (works now that a RateGroup exists).
4. Inside **each zone** (United States + Europe + any extra), click **New Rate** and create **exactly these two rates with NO conditions**:

   | Rate Name | Price | Transit Time | Conditions |
   |---|---|---|---|
   | `Standard` | `0` (free) | e.g. 3-5 days | **Leave blank — no conditions** |
   | `Fast` | `5` | e.g. 1-2 days | **Leave blank — no conditions** |

   - Currency must match the WebStore's currency (see D.5 — typically `EUR` for Spain demos, `USD` for US demos).
   - Save each rate. The `ShippingCarrierMethod` is created automatically behind the scenes.
   - **Do NOT add condition rules** (weight ranges, order totals, etc.) — leaving conditions blank ensures the rate matches every cart unconditionally. If a rate has a condition that the cart doesn't satisfy, checkout shows zero methods and breaks.

5. In the **Products** section of the profile, click **Manage** and either:
   - Assign a Product2 from your catalog that acts as the "shipping charge product" (any simple active Product2 will do — it's what the engine uses to bill the shipping line), OR
   - Move the existing shipping-charge product from the cloned SDO profile (see **cleanup** below).

### D.4 — Post-API cleanup (delete SDO ghosts)

After cloning, check for `ShippingConfigSetProduct` rows still pointing to SDO shipping profiles (profile `TargetRecordId != YourWebStoreId`):

```text
sf data query -q "SELECT Id, ShippingProfileId, Product2Id, Product2.Name FROM ShippingConfigSetProduct" -o <alias>
```

Any row whose `ShippingProfileId` belongs to an SDO store (not your new one) is a **ghost from the clone** and must be deleted, otherwise shipping for those products resolves against a non-existent zone and breaks checkout with `"can't deliver this item"`:

```text
sf data delete record -s ShippingConfigSetProduct -i <GHOST_ID> -o <alias>
```

### D.5 — Currency × Country matrix

`StandardShipment` matches rates by `ShippingRateArea.Countries` **and** `StandardShippingRate.CurrencyIsoCode` — both must align with the cart. If the store has multiple `SupportedCurrencies`, you need **one full set of rates per currency per zone**, otherwise the buyer sees no methods when their cart is in the "wrong" currency.

Practical recommendations:

- For demo stores, **set `WebStore.SupportedCurrencies` to a single currency** matching the target buyer persona (e.g. `EUR` for a Spain demo). Update the WebStore via Apex if it was cloned with `USD;EUR`:

  ```apex
  WebStore ws = [SELECT Id, SupportedCurrencies FROM WebStore WHERE Id='<WebStore_Id>'];
  ws.SupportedCurrencies = 'EUR';
  update ws;
  ```

- If you keep multiple currencies, duplicate every rate in every currency. A 2-country × 2-currency × 3-speed matrix = 12 rates — all manual via UI.

### D.6 — Debugging tips when checkout fails at shipping

- `CartValidationOutput` with message `"No Shipping Rates are configured for this selection for the web Store <Id>"` → the default profile has no rate matching the delivery country **+ currency** combo → return to D.3/D.5.
- `CartValidationOutput` never appears **and** the 400 response mentions `Region Code is not supported` → the rate's carrier method is missing (rate created via API with `ShippingCarrierMethodId=null`). Delete the API-created rate and re-create it via D.3 UI.
- Browser console shows `409 Conflict` or `CHECKOUT_FORM_BUSY` during retries → there are `WebCart` rows stuck in `Status='Checkout'` **and/or** orphaned `CartCheckoutSession` rows from previous failed attempts. Nuke both:

  ```apex
  List<WebCart> open = [SELECT Id FROM WebCart WHERE WebStoreId='<Id>' AND Status='Checkout'];
  for (WebCart c : open) c.Status='Closed';
  update open;
  delete [SELECT Id FROM CartCheckoutSession];
  ```

- Even after clearing DB state, a **logged-in browser session persists LWR state in `localStorage`** that points to the now-deleted `CartCheckoutSession` → the LWR app retries the dead session and surfaces `CHECKOUT_FORM_BUSY`. Fix by either:
  - DevTools → Application → Storage → **Clear site data** for the store's `*.my.site.com` origin, then log in again, OR
  - Test from a fresh incognito window (always works because `localStorage` is empty).
  - Guest/incognito sessions never hit this because they don't persist state.

## Phase E — Buyer groups, buyer accounts, contacts, store association

### Default buyer layout (mandatory for every new store)

Every new WebStore created with this skill **must** ship with **two buyer groups, two buyer accounts and two contacts** — one Standard, one VIP — wired to the three pricebooks from Phase B. This is the deterministic baseline; add more groups/accounts later if needed, but never skip these two.

| Layer | Standard tier | VIP tier |
|---|---|---|
| `BuyerGroup` | `<Site Name> Standard Customers` | `<Site Name> VIP Customers` |
| `Account` | `<Site Name> Standard Buyer Account` | `<Site Name> VIP Buyer Account` |
| `BuyerAccount` | `BuyerId = Standard Account.Id`, `BuyerStatus=Active`, `CommerceType=Buyer`, **then update `IsActive=true`** | `BuyerId = VIP Account.Id`, `BuyerStatus=Active`, `CommerceType=Buyer`, **then update `IsActive=true`** |
| `Contact` | First/Last name `Standard Buyer`, valid email | First/Last name `VIP Buyer`, valid email |
| `BuyerGroupPricebook` | Sale PB `Priority=1`, **List** PB `Priority=2` (strikethrough) | VIP PB `Priority=1`, **List** PB `Priority=2` (strikethrough) |
| `CommerceEntitlementBuyerGroup` | Same `CommerceEntitlementPolicy` as the rest of the store | Same `CommerceEntitlementPolicy` as the rest of the store |

> **Why two pricebooks per group with explicit `Priority`?** The pricing engine returns `Price Unavailable` whenever a buyer group has more than one active `BuyerGroupPricebook` row with `Priority=null`. Setting `Priority=1` on the offer pricebook (Sale or VIP) and `Priority=2` on the List one disambiguates the lookup and gives the storefront a strikethrough out of the box.

### One-shot bootstrap commands (ordered)

Resolve `<WEBSTORE_ID>`, `<LIST_PB_ID>` (the **custom** List Price Book created in Phase B — never the org Standard, which Salesforce rejects on `BuyerGroupPricebook`), `<SALE_PB_ID>`, `<VIP_PB_ID>`, `<POLICY_ID>` (your `CommerceEntitlementPolicy`) and `<ACCOUNT_RECORD_TYPE_ID>` first.

```text
# 1. Buyer groups (create both, capture IDs as <STD_BG_ID> and <VIP_BG_ID>)
sf data create record -s BuyerGroup -v "Name='<Site Name> Standard Customers' Description='Default buyer group for standard pricing.'" -o <alias>
sf data create record -s BuyerGroup -v "Name='<Site Name> VIP Customers'      Description='Buyer group for VIP pricing tier.'"      -o <alias>

# 2. Attach both groups to the WebStore
sf data create record -s WebStoreBuyerGroup -v "WebStoreId=<WEBSTORE_ID> BuyerGroupId=<STD_BG_ID>" -o <alias>
sf data create record -s WebStoreBuyerGroup -v "WebStoreId=<WEBSTORE_ID> BuyerGroupId=<VIP_BG_ID>" -o <alias>

# 3. Entitlements — both groups see the catalog/prices through the same policy
sf data create record -s CommerceEntitlementBuyerGroup -v "BuyerGroupId=<STD_BG_ID> PolicyId=<POLICY_ID>" -o <alias>
sf data create record -s CommerceEntitlementBuyerGroup -v "BuyerGroupId=<VIP_BG_ID> PolicyId=<POLICY_ID>" -o <alias>

# 4. Pricebook wiring (Sale/VIP at Priority=1, List at Priority=2 for strikethrough).
#    NEVER use the org Standard Price Book here — Salesforce rejects it with
#    "Standard Price Book isn't available to stores or buyer groups."
sf data create record -s BuyerGroupPricebook -v "BuyerGroupId=<STD_BG_ID> Pricebook2Id=<SALE_PB_ID> IsActive=true Priority=1" -o <alias>
sf data create record -s BuyerGroupPricebook -v "BuyerGroupId=<STD_BG_ID> Pricebook2Id=<LIST_PB_ID> IsActive=true Priority=2" -o <alias>
sf data create record -s BuyerGroupPricebook -v "BuyerGroupId=<VIP_BG_ID> Pricebook2Id=<VIP_PB_ID>  IsActive=true Priority=1" -o <alias>
sf data create record -s BuyerGroupPricebook -v "BuyerGroupId=<VIP_BG_ID> Pricebook2Id=<LIST_PB_ID> IsActive=true Priority=2" -o <alias>

# 5. Standard buyer — Account, BuyerAccount, Contact
sf data create record -s Account      -v "Name='<Site Name> Standard Buyer Account' RecordTypeId=<ACCOUNT_RECORD_TYPE_ID>" -o <alias>
sf data create record -s BuyerAccount -v "BuyerId=<STD_ACCOUNT_ID> Name='<Site Name> Standard Buyer Account' BuyerStatus=Active CommerceType=Buyer" -o <alias>
sf data create record -s Contact      -v "FirstName='Standard' LastName='Buyer' Email='standard.buyer@example.com' AccountId=<STD_ACCOUNT_ID>" -o <alias>

# 6. VIP buyer — same shape, different tier
sf data create record -s Account      -v "Name='<Site Name> VIP Buyer Account' RecordTypeId=<ACCOUNT_RECORD_TYPE_ID>" -o <alias>
sf data create record -s BuyerAccount -v "BuyerId=<VIP_ACCOUNT_ID> Name='<Site Name> VIP Buyer Account' BuyerStatus=Active CommerceType=Buyer" -o <alias>
sf data create record -s Contact      -v "FirstName='VIP' LastName='Buyer' Email='vip.buyer@example.com' AccountId=<VIP_ACCOUNT_ID>" -o <alias>

# 7. CRITICAL — Activate BuyerAccount so Account.IsBuyer flips to true.
#    Inserting BuyerAccount with BuyerStatus=Active is NOT enough; you must
#    update IsActive=true in a separate DML statement. Without this step the
#    next BuyerGroupMember insert fails with:
#       "This Account isn't a Buyer Account. Enable this Account as a buyer and try again."
sf data update record -s BuyerAccount -i <STD_BUYER_ACCOUNT_ID> -v "IsActive=true" -o <alias>
sf data update record -s BuyerAccount -i <VIP_BUYER_ACCOUNT_ID> -v "IsActive=true" -o <alias>

# 8. Now (and only now) BuyerGroupMember accepts the BuyerId
sf data create record -s BuyerGroupMember -v "BuyerGroupId=<STD_BG_ID> BuyerId=<STD_ACCOUNT_ID>" -o <alias>
sf data create record -s BuyerGroupMember -v "BuyerGroupId=<VIP_BG_ID> BuyerId=<VIP_ACCOUNT_ID>" -o <alias>

# 9. CRITICAL — link the site's Guest Buyer Profile to the Standard buyer group.
#    Without this, anonymous visitors see the catalog mega-menu (Phase A guest
#    enablement) but every product renders "Price Unavailable", because the
#    pricing engine has no buyer-group → pricebook chain to resolve for guests.
#    GuestBuyerProfile is polymorphic on BuyerGroupMember.BuyerId, identical
#    pattern to the Account-based members above. Resolve the site's auto-created
#    GuestBuyerProfile.Id from the WebStore first.
sf data query -q "SELECT GuestBuyerProfileId FROM WebStore WHERE Id = '<WEBSTORE_ID>'" -o <alias>
# capture <SITE_GUEST_BUYER_PROFILE_ID>
sf data create record -s BuyerGroupMember -v "BuyerGroupId=<STD_BG_ID> BuyerId=<SITE_GUEST_BUYER_PROFILE_ID>" -o <alias>
```

> **Why the Standard buyer group (and not a separate Guest market group)?** Both options work — the Standard group is preferred for demos because it ships with the **Sale** pricebook at `Priority=1` and the **List** pricebook at `Priority=2` for strikethrough, exactly the dual-price experience the storefront is designed to show. Linking the guest profile here means anonymous visitors see the same Sale pricing as logged-in Standard buyers, which is the correct out-of-the-box demo behavior. If the demo specifically needs guests to see **list** prices only (no discount), create a third Market-role buyer group instead and link the guest profile to that one — but never leave the guest profile with no buyer group, which is the parked failure mode.

> **Why the extra `IsActive=true` step?** `Account.IsBuyer` is a system-managed read-only field — neither Data API nor Apex can write it (`Field is not writeable: Account.IsBuyer` / `Unable to create/update fields: IsBuyer`). The platform flips it automatically when a `BuyerAccount` row pointing at the account is set to `IsActive=true`. The `BuyerStatus=Active` field set during insert is **not** the trigger. Verify with `SELECT IsBuyer FROM Account WHERE Id IN ('<STD_ACCOUNT_ID>','<VIP_ACCOUNT_ID>')` — both must come back `true` before step 8.

> Login users for these two contacts and the matching permission set assignments are **created automatically** in **Phase F** — never leave a new store at this step.

### Verification queries (run all four)

```text
sf data query -q "SELECT Name, Id FROM BuyerGroup WHERE Id IN ('<STD_BG_ID>','<VIP_BG_ID>')" -o <alias>
sf data query -q "SELECT BuyerGroup.Name, Pricebook2.Name, IsActive, Priority FROM BuyerGroupPricebook WHERE BuyerGroupId IN ('<STD_BG_ID>','<VIP_BG_ID>') ORDER BY BuyerGroup.Name, Priority" -o <alias>
sf data query -q "SELECT Account.Name, BuyerStatus, CommerceType FROM BuyerAccount WHERE BuyerId IN ('<STD_ACCOUNT_ID>','<VIP_ACCOUNT_ID>')" -o <alias>
sf data query -q "SELECT BuyerGroup.Name, Buyer.Name FROM BuyerGroupMember WHERE BuyerGroupId IN ('<STD_BG_ID>','<VIP_BG_ID>')" -o <alias>
```

Expected: 2 buyer groups, 4 `BuyerGroupPricebook` rows (Priority 1 + 2 per group), 2 active `BuyerAccount` rows, 2 `BuyerGroupMember` rows.

### Reference (legacy quick recipe)

For ad-hoc / extra buyer groups beyond the default Standard + VIP pair, the underlying object recipe is:

1. **`BuyerGroup`**: create (Name only required).
2. **`WebStoreBuyerGroup`**: `WebStoreId` + `BuyerGroupId`.
3. **`Account`**: create (record type per org; SDO buyer accounts often use **SDO_Account_Simple**).
4. **`BuyerAccount`**: **required** so **`BuyerGroupMember`** accepts `BuyerId`. Two-step pattern is mandatory:
   - **a.** Insert with `BuyerId = Account.Id`, `BuyerStatus=Active`, `CommerceType=Buyer`.
   - **b.** **Update** the new `BuyerAccount` with `IsActive=true` in a **separate** DML. Only this update flips `Account.IsBuyer=true`; without it, every subsequent `BuyerGroupMember` insert fails with *"This Account isn't a Buyer Account."* `Account.IsBuyer` itself is read-only (cannot be set via Data API or Apex), and `BuyerStatus=Active` alone does **not** trigger the flip.
5. **`Contact`**: under that Account (login identity).
6. **`BuyerGroupMember`**: `BuyerGroupId` + `BuyerId` (**Account Id**).
7. **`BuyerGroupPricebook`**: mirror SDO’s mapping from buyer groups to **`Pricebook2`** where applicable (same **`Pricebook2`** IDs as the reference store if parity is required).
8. **`CommerceEntitlementBuyerGroup`** (**required for catalog + pricing visibility**): link each **`BuyerGroup`** used by the store to the same **`CommerceEntitlementPolicy`** as the reference B2B store (often **“Cirrus Entitlement Policy”** in SDO packs). Without this, buyers can see PLP/PDP but **no prices** (entitlement gate).

```text
sf data query -q "SELECT BuyerGroupId, PolicyId FROM CommerceEntitlementBuyerGroup WHERE BuyerGroupId IN (SELECT BuyerGroupId FROM WebStoreBuyerGroup WHERE WebStoreId = '<SDO_WEBSTORE_ID>')" -o <alias>

sf data create record -s CommerceEntitlementBuyerGroup -o <alias> -v "BuyerGroupId=<YOUR_BUYER_GROUP_ID> PolicyId=<POLICY_ID_FROM_REFERENCE>"
```

**Do not clone** **`WebStoreBuyerGroup`** rows that point at **SDO-owned buyer groups** into a second **`WebStore`** if the platform enforces **one store per market** — you may get `You can't associate a market to more than one store.` Create **new buyer groups** for the new store and copy **entitlements + price books** instead.

**Guests (unauthenticated) and Market buyer groups:** Account-based buyer groups (**`Role = AccountBased`**) never apply to guests—guests have no buyer account. Use a **`BuyerGroup`** with **`Role = Market`** (same pattern as **North America** in SDO): attach **`WebStoreBuyerGroup`**, **`BuyerGroupPricebook`**, **`CommerceEntitlementBuyerGroup`**, then **`BuyerGroupBuyerCriteria`** rows that reference existing **`BuyerCriteria`** records (e.g. **`SessionAttributes` / `Locale` / `en_US`**) shared with another market group. **Do not** create duplicate **`BuyerCriteria`** rows for the same locale—the org enforces uniqueness; **reuse** the same **`BuyerCriteria.Id`** with a **new** **`BuyerGroupBuyerCriteria`** row pointing at your Market **`BuyerGroup`**.

**Guest Buyer Profile ↔ buyer groups (Related list parity):** When **`WebStore.GuestBuyerProfileId`** stays on a **template** **`GuestBuyerProfile`** (Commerce **Guest Access** created or rename-only — cannot pick **`B2B Guest Buyer Profile`**), mirror SDO’s **`BuyerGroupMember`** rows from **`B2B Guest Buyer Profile`** onto your template profile. **`BuyerGroupMember.BuyerId`** is polymorphic (**`Account`** or **`GuestBuyerProfile`**).

```text
sf data query -q "SELECT BuyerGroupId FROM BuyerGroupMember WHERE BuyerId = '<REFERENCE_GUEST_BUYER_PROFILE_ID>'" -o <alias>
sf data create record -s BuyerGroupMember -o <alias> -v "BuyerGroupId=<GROUP_ID> BuyerId=<YOUR_TEMPLATE_GUEST_BUYER_PROFILE_ID>"
```

Example SDO pack: **`Cirrus Buyer Group`** (**`AccountBased`**) linked to **`B2B Guest Buyer Profile`** — replicate that **`BuyerGroupMember`** row for **`Cursor Commerce Demo Guest Buyer Profile`** when needed.

## Phase F — Experience Cloud login + permission sets (fully automated, mandatory)

**Hard rule for the agent:** every new store **must** finish Phase F before being handed back to the user. Never leave permission set assignment, user creation, or `Network` flags as "for you to do later" — the manual catalogue at `USER_MANUAL_STEPS.md` only lists items the API genuinely blocks.

This phase has **four blocks**. Run them in order; they are all CLI/Apex driven.

### F.1 — Network member groups (profile + permission sets allowed on the site)

The site's **Members** UI maps to `NetworkMemberGroup`. Add the buyer profile and the Commerce permission sets so freshly-created buyer Users can actually log in, and so the Site Guest User picks up the storefront permission sets via profile.

```text
# Resolve the IDs once
sf data query -q "SELECT Id FROM Profile WHERE Name = 'B2B Reordering Portal Buyer Profile'" -o <alias>
sf data query -q "SELECT Id, Name FROM PermissionSet WHERE Name IN ('B2B_Commerce_Guest_Browser_Access','B2BCommerce_Community_Access','D2C_Commerce_Guest_User_Access','B2BBuyer','B2B_Commerce_Customer_Community_Plus_Access')" -o <alias>

# Allow the buyer profile on the site
sf data create record -s NetworkMemberGroup -v "NetworkId=<NETWORK_ID> ParentId=<B2B_BUYER_PROFILE_ID>" -o <alias>

# Allow the Commerce permission sets on the site (one row per PS)
sf data create record -s NetworkMemberGroup -v "NetworkId=<NETWORK_ID> ParentId=<B2B_Commerce_Guest_Browser_Access_ID>" -o <alias>
sf data create record -s NetworkMemberGroup -v "NetworkId=<NETWORK_ID> ParentId=<B2BCommerce_Community_Access_ID>"      -o <alias>
sf data create record -s NetworkMemberGroup -v "NetworkId=<NETWORK_ID> ParentId=<D2C_Commerce_Guest_User_Access_ID>"   -o <alias>
```

Avoid duplicates by querying first:

```text
sf data query -q "SELECT ParentId FROM NetworkMemberGroup WHERE NetworkId = '<NETWORK_ID>'" -o <alias>
```

### F.2 — Site Guest User permission sets + currency (anonymous catalog access)

The Site Guest User (auto-created with the site, profile usually `<Site Name> Profile`) must have the same Commerce guest permission sets the SDO Guest User has, otherwise the storefront menu and PLP render empty for anonymous visitors.

```text
sf data query -q "SELECT Id FROM User WHERE Profile.Name = '<Site Name> Profile' LIMIT 1" -o <alias>
# capture <SITE_GUEST_USER_ID>

sf data create record -s PermissionSetAssignment -v "AssigneeId=<SITE_GUEST_USER_ID> PermissionSetId=<B2B_Commerce_Guest_Browser_Access_ID>" -o <alias>
sf data create record -s PermissionSetAssignment -v "AssigneeId=<SITE_GUEST_USER_ID> PermissionSetId=<D2C_Commerce_Guest_User_Access_ID>"   -o <alias>
```

**CRITICAL — Guest User currency must match Phase 0 Q4 `<CURRENCY>`**. New site Guest Users inherit the org's corporate currency regardless of WebStore or Network settings. If `<CURRENCY>` ≠ corporate currency, anonymous visitors see **"Price Unavailable"** on every product because the pricing engine filters `PricebookEntry` by `User.DefaultCurrencyIsoCode`. Update the Guest User in the same phase you assign its permission sets — multi-currency orgs only:

```text
sf data update record -s User -i <SITE_GUEST_USER_ID> -v "DefaultCurrencyIsoCode=<CURRENCY>" -o <alias>
```

Also align the Guest User's language/locale with the WebStore's `DefaultLanguage` (Phase A) so labels render correctly for anonymous visitors. Example for a Spanish EUR storefront:

```text
sf data update record -s User -i <SITE_GUEST_USER_ID> -v "DefaultCurrencyIsoCode=EUR LanguageLocaleKey=es LocaleSidKey=es_ES" -o <alias>
```

Verify with:

```text
sf data query -q "SELECT DefaultCurrencyIsoCode, LanguageLocaleKey, LocaleSidKey FROM User WHERE Id = '<SITE_GUEST_USER_ID>'" -o <alias>
sf data query -q "SELECT PermissionSet.Name FROM PermissionSetAssignment WHERE AssigneeId = '<SITE_GUEST_USER_ID>'" -o <alias>
```

### F.3 — Buyer Users for the Standard + VIP contacts (Apex insert + permission sets)

`sf org create user` only works on scratch orgs. For sandboxes/DX orgs use **Apex** so the agent owns the whole flow end-to-end. Create a single anonymous Apex script that inserts both Users (one per contact created in Phase E), assigns the three buyer permission sets (`B2BBuyer`, `B2BCommerce_Community_Access`, `B2B_Commerce_Customer_Community_Plus_Access`), and sets a deterministic temporary password so the user can immediately test the login.

Save as `scripts/apex/create-buyer-users.apex` and run with `sf apex run --file scripts/apex/create-buyer-users.apex --target-org <alias>`:

```apex
// One-shot script — creates Users for the Standard + VIP contacts and assigns buyer PSAs.
// Substitute the literal IDs/usernames for your store before running.
String stdContactId   = '<STD_CONTACT_ID>';
String vipContactId   = '<VIP_CONTACT_ID>';
String siteSuffix     = '<unique-suffix>';                  // e.g. cursor-commerce-demo
String tempPassword   = 'Buyer!Demo123';                    // rotate after first login

Id buyerProfileId = [
    SELECT Id FROM Profile WHERE Name = 'B2B Reordering Portal Buyer Profile' LIMIT 1
].Id;

Map<String, Id> psByName = new Map<String, Id>();
for (PermissionSet ps : [
    SELECT Id, Name FROM PermissionSet
    WHERE Name IN ('B2BBuyer','B2BCommerce_Community_Access','B2B_Commerce_Customer_Community_Plus_Access')
]) {
    psByName.put(ps.Name, ps.Id);
}

// <CURRENCY> and <LOCALE> come from Phase 0 Q4 + Q2. For a Spanish EUR storefront
// use currency='EUR', locale='es_ES', language='es'. For a US USD storefront use
// currency='USD', locale='en_US', language='en_US'. On single-currency orgs the
// DefaultCurrencyIsoCode field does not exist — remove that line before running.
String currencyIso = '<CURRENCY>';    // e.g. 'EUR', 'USD', 'GBP'
String localeKey   = '<LOCALE>';      // e.g. 'es_ES', 'en_US'
String languageKey = '<LANGUAGE>';    // e.g. 'es', 'en_US'

List<User> toInsert = new List<User>{
    new User(
        ContactId              = stdContactId,
        ProfileId              = buyerProfileId,
        Username               = 'standard.buyer@' + siteSuffix + '.demo',
        Email                  = 'standard.buyer@example.com',
        FirstName              = 'Standard',
        LastName               = 'Buyer',
        Alias                  = 'stdbuy',
        TimeZoneSidKey         = 'Europe/Madrid',
        LocaleSidKey           = localeKey,
        EmailEncodingKey       = 'UTF-8',
        LanguageLocaleKey      = languageKey,
        DefaultCurrencyIsoCode = currencyIso
    ),
    new User(
        ContactId              = vipContactId,
        ProfileId              = buyerProfileId,
        Username               = 'vip.buyer@' + siteSuffix + '.demo',
        Email                  = 'vip.buyer@example.com',
        FirstName              = 'VIP',
        LastName               = 'Buyer',
        Alias                  = 'vipbuy',
        TimeZoneSidKey         = 'Europe/Madrid',
        LocaleSidKey           = localeKey,
        EmailEncodingKey       = 'UTF-8',
        LanguageLocaleKey      = languageKey,
        DefaultCurrencyIsoCode = currencyIso
    )
};
insert toInsert;

List<PermissionSetAssignment> psas = new List<PermissionSetAssignment>();
for (User u : toInsert) {
    for (Id psId : psByName.values()) {
        psas.add(new PermissionSetAssignment(AssigneeId = u.Id, PermissionSetId = psId));
    }
}
insert psas;

for (User u : toInsert) {
    System.setPassword(u.Id, tempPassword);
}

System.debug('Created users: ' + toInsert);
```

After running, verify:

```text
sf data query -q "SELECT Id, Username, IsActive, ContactId FROM User WHERE ContactId IN ('<STD_CONTACT_ID>','<VIP_CONTACT_ID>')" -o <alias>
sf data query -q "SELECT Assignee.Username, PermissionSet.Name FROM PermissionSetAssignment WHERE AssigneeId IN (SELECT Id FROM User WHERE ContactId IN ('<STD_CONTACT_ID>','<VIP_CONTACT_ID>')) ORDER BY Assignee.Username, PermissionSet.Name" -o <alias>
```

Expected: 2 active Users, 6 `PermissionSetAssignment` rows total (3 PS × 2 users).

> **Self-registration handler parity:** if the org uses a custom `CommunitiesSelfRegConfig` Apex class or a Login/Registration Flow, paste the same `PermissionSetAssignment` block into that handler so every newly-registered buyer gets the three PS automatically. The agent should patch the existing handler when present; only flag it as TODO if no handler exists yet.

### F.4 — Network flags required for Login-as and self-registration

```text
sf data update record -s Network -i <NETWORK_ID> -v "OptionsAllowInternalUserLogin=true OptionsSelfRegistrationEnabled=true OptionsEmbeddedLoginEnabled=true" -o <alias>
sf data update record -s Network -i <NETWORK_ID> -v "SelfRegProfileId=<B2B_BUYER_PROFILE_ID>" -o <alias>
sf community publish --name '<Site Name>' --target-org <alias>
```

After this phase the Standard and VIP buyers can log in directly with `standard.buyer@<suffix>.demo` / `vip.buyer@<suffix>.demo` + the temporary password, and admin **Login as** lands authenticated on the storefront.

## Phase F2 — Branding (logo, colors, hero copy) via DigitalExperienceBundle

All B2B Commerce LWR branding lives **inside the site bundle**, no Builder-only steps needed:

| File (under `digitalExperiences/site/<SITE_BUNDLE>/`) | Purpose |
|----|----|
| `sfdc_cms__brandingSet/<SET>/content.json` | Color tokens (`PrimaryAccentColor`, `_PrimaryAccentColor1..3`, `TextColor`, button colors, `LinkColor`), `BaseFont`, `SiteLogo` (must be a **server-relative path**, not an external URL — deploy fails with `None of the rules validated property: SiteLogo`). To swap the logo, deploy the new image as a **`StaticResource`** and set both `SiteLogo` and `_SiteLogoUrl` to the LWR-safe path **`/sfsites/c/resource/<ResourceName>`** (see *External assets* below). The shorter `/resource/<ResourceName>` path may deploy and the resource may return `200`, but seeded LWR Commerce headers can fail to render it. The CSS `styles.css` cannot reliably reach the logo `<img>` because the `dxp_content_layout:siteLogo` LWC renders it inside a shadow DOM. **Each Commerce Store (LWR) bundle ships with FOUR brandingSets — `B2B_Commerce` (main), `B2B_Footer`, `B2B_Home_Banner`, `B2B_Right_Panel`; seeded SDO bundles may include additional sets such as `Home_Header`. Apply the color tokens (and `LinkColor`, `TextColor`) to every branding set present; only the main commerce branding set carries the `SiteLogo` keys.** Otherwise the footer / right panel / home banner keep the template's stock colors. |
| `sfdc_cms__themeLayout/commerceLayout/content.json` (and `myAccountLayout`) | Header / footer markup. The top promo bar is a `richTextValue` HTML string (`<div style="background-color:#1C1C1C">…</div>`) — search by visible copy ("Exclusive winter sale" in the Cursor template) and replace HTML in place. |
| `sfdc_cms__view/home/content.json` (+ `tablet/tablet.json`, `mobile/mobile.json`) | Hero text, button labels, **hero/banner image references (MANDATORY for client branding — see below)**. The Cursor LWR template ships stock banner images referenced in `imageInfo`: `assets/images/home-banner-2.jpg` ("Main Pods Banner", left/big hero, ~1920×600), `assets/images/homeBanner-right.png` ("Quick Order Pods Banner", right secondary, ~960×450), and seeded SDO tiles such as `emergency-prep-b2b.jpg`. **Every stock/SDO URL MUST be replaced with brand-relevant external URLs** — otherwise the homepage keeps showing template or Cirrus content even after colors/copy are rebranded. The template's `tablet.json` and `mobile.json` do **not** carry `imageInfo` of their own; they inherit from `content.json`. Replace plain-text strings (e.g. `"Unleash Your Inner Power…"`, `"Emergency Prep"`, `"Discover Nature's Energy"`). |
| `sfdc_cms__site/<SITE>/content.json` | `title` (browser tab + share metadata), `contentBody.authenticationType`. **Strip `geoBotsAllowed` before deploy — see warning below.** |
| `sfdc_cms__styles/styles_css/styles.css` | Free-form CSS appended **after** the SLDS + `theme2.css` defaults. Best place to override DXP custom properties (`--dxp-g-brand`, `--dxp-g-link`, `--dxp-g-primary`) and to swap the logo via CSS `content: url("https://…")` (preferred when the host is whitelisted) or `content: url("data:image/svg+xml;base64,…")` as a fully self-contained fallback. |

> **CRITICAL — strip `geoBotsAllowed` after retrieve.** `sf project retrieve start` for a `Commerce Store (LWR)` bundle pulls down `sfdc_cms__site/<SITE>/content.json` containing a `geoBotsAllowed` property. The deploy schema for the same metadata type rejects this property and the whole bundle deploy fails with:
> ```
> [{"propertyPath":"$.geoBotsAllowed","message":"You can't add the geoBotsAllowed property defined at $.geoBotsAllowed because the `additionalProperties` keyword value is set to false."}]
> ```
> Always remove it from `sfdc_cms__site/<SITE>/content.json` (and any nested copies) **before** running `sf project deploy start`. A one-liner that walks the JSON works: `python3 -c "import json,sys; p=sys.argv[1]; d=json.load(open(p)); def s(n): (isinstance(n,dict) and (n.pop('geoBotsAllowed',None) or [s(v) for v in n.values()])) or (isinstance(n,list) and [s(x) for x in n]); s(d); json.dump(d, open(p,'w'), indent=2)" force-app/main/default/digitalExperiences/site/<SITE>/sfdc_cms__site/<SITE>/content.json`

Workflow:

```text
mkdir branding-work && cd branding-work
sf project generate --name . --output-dir . --template empty
sf project retrieve start --target-org <alias> --metadata "DigitalExperienceBundle:site/<SITE_BUNDLE>"

# 1. Strip the deploy-breaking property
python3 -c "
import json, pathlib
p = pathlib.Path('force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/sfdc_cms__site/<SITE_BUNDLE>/content.json')
d = json.loads(p.read_text())
def strip(n):
    if isinstance(n, dict):
        n.pop('geoBotsAllowed', None)
        for v in list(n.values()): strip(v)
    elif isinstance(n, list):
        for x in n: strip(x)
strip(d)
p.write_text(json.dumps(d, indent=2))
"

# 2. Edit JSON/CSS files (apply color tokens to ALL FOUR brandingSets, banner HTML, hero copy, styles.css)
# 3. Replace homepage banners (MANDATORY — never leave stock Cursor/SDO/Cirrus content on a client demo)
#    - In sfdc_cms__view/home/content.json, find both occurrences of:
#         "imageInfo": "{\"imageInfoV1\":{\"url\":\"assets/images/home-banner-2.jpg\",...}}"
#         "imageInfo": "{\"imageInfoV1\":{\"url\":\"assets/images/homeBanner-right.png\",...}}"
#      and any seeded SDO/Cirrus image URLs such as:
#         "https://sfdc-ckz-b2b.s3.amazonaws.com/SDO/Commerce_Images/emergency-prep-b2b.jpg"
#      Rewrite each inner "url" to a brand-relevant external image (e.g. a hero or product
#      image from the client's marketing/catalog site). Update altText to a meaningful
#      description and change visible titles such as "Emergency Prep".
#    - The brand domain MUST already be a CspTrustedSite with isApplicableToImgSrc=true
#      (deployed in the StaticResource / CSP step above).
#    - Verify each new URL returns HTTP 200 with image/* Content-Type before deploying:
#         curl -sI -A "Mozilla/5.0" "<image_url>" | head -5
#    - Pick a wide image (>=1920px) for the main hero (renders ~1920x600) and a
#      portrait/landscape image (>=960px) for the right panel (renders ~960x450).
#    - Also search for custom home components such as c:parallaxcmp and replace stock
#      titles like "Discover Nature's Energy" with brand-appropriate CSR/sustainability
#      copy (for example "Committed to Sustainable Oral Health" for Dentaid).
# 4. Deploy + publish
sf project deploy start --target-org <alias> --source-dir force-app/main/default/digitalExperiences/site/<SITE_BUNDLE> --ignore-conflicts
sf community publish --name '<Site Name>' --target-org <alias>
```

> **Bundle name has a trailing `1`.** When the site is created via `sf community create`, the resulting `DigitalExperienceBundle` is named `<SiteNameWithUnderscores>1` (e.g. site `Ascendum Commerce Demo` → bundle `Ascendum_Commerce_Demo1`). Discover the exact name with `sf org list metadata --metadata-type DigitalExperienceBundle --target-org <alias> | grep <SiteNamePartial>` before retrieving.

> **`sf project deploy start --source-dir force-app/main/default/digitalExperiences` deploys EVERY bundle present in that folder.** If the workspace already has bundles from previous store builds (e.g. `Cursor_Commerce_Demo1`), they will all be redeployed. Use `--source-dir` pointed at the specific bundle folder to limit scope: `--source-dir force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>`.

### External assets (logo + hero images) via CSP Trusted URLs

`SiteLogo` rejects external URLs, but **CSS `content: url(...)` and hero `imageInfo.url` both accept any host** that the LWR site’s **CSP allow-list** trusts. Add the asset domain as a **`CspTrustedSite`** with all relevant directives, then reference the asset everywhere by full URL — no base64, no copying images into the bundle.

Deploy a CSP Trusted Site for the brand CDN (one per domain):

```xml
<!-- force-app/main/default/cspTrustedSites/Ascendum_CDN.cspTrustedSite-meta.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<CspTrustedSite xmlns="http://soap.sforce.com/2006/04/metadata">
    <description>Ascendum CDN - branding assets (logo + hero images).</description>
    <endpointUrl>https://ascendum.pt</endpointUrl>
    <isActive>true</isActive>
    <context>All</context>
    <isApplicableToConnectSrc>true</isApplicableToConnectSrc>
    <isApplicableToFontSrc>true</isApplicableToFontSrc>
    <isApplicableToFrameSrc>true</isApplicableToFrameSrc>
    <isApplicableToImgSrc>true</isApplicableToImgSrc>
    <isApplicableToMediaSrc>true</isApplicableToMediaSrc>
    <isApplicableToStyleSrc>true</isApplicableToStyleSrc>
</CspTrustedSite>
```

```text
sf project deploy start --target-org <alias> --source-dir force-app/main/default/cspTrustedSites
```

**Logo replacement (StaticResource — required LWR path).** The `dxp_content_layout:siteLogo` component lives inside an LWC shadow DOM, so `styles.css` cannot reliably reach the `<img>`. Deploy the new logo as a `StaticResource` and rewrite `SiteLogo` + `_SiteLogoUrl` to `/sfsites/c/resource/<ResourceName>`, matching the path style used by seeded SDO Commerce bundles.

```text
mkdir -p force-app/main/default/staticresources
curl -s -o force-app/main/default/staticresources/AscendumLogo.svg \
  https://ascendum.pt/wp-content/themes/ascendum_pt_theme/assets/imgs/logos/Ascendum_Wordmark-02_Branco.svg
cat > force-app/main/default/staticresources/AscendumLogo.resource-meta.xml <<'XML'
<?xml version="1.0" encoding="UTF-8"?>
<StaticResource xmlns="http://soap.sforce.com/2006/04/metadata">
    <cacheControl>Public</cacheControl>
    <contentType>image/svg+xml</contentType>
</StaticResource>
XML
sf project deploy start --target-org <alias> --source-dir force-app/main/default/staticresources
```

Then in `sfdc_cms__brandingSet/B2B_Commerce/content.json` (only this set carries the logo keys; the other three brandingSets `B2B_Footer`, `B2B_Home_Banner`, `B2B_Right_Panel` do not have `SiteLogo` / `_SiteLogoUrl`):

```json
"SiteLogo": "/sfsites/c/resource/AscendumLogo",
"_SiteLogoUrl": "url(/sfsites/c/resource/AscendumLogo)"
```

Re-deploy the bundle and republish. Verify the published site can load `https://<site-domain>/<site-path>/sfsites/c/resource/<ResourceName>` with `content-type: image/*` or `image/svg+xml`. (Use a wide wordmark SVG — `viewBox="0 0 200 44"` shape — so it fills the header strip; the small mark-only file `Ascendum_Wordmark-05_Branco.svg` has `viewBox="0 0 140 130"` and renders as a tiny square.)

**Watch the contrast.** The Cursor LWR template ships a **white header background**. If the only public version of the brand wordmark is white-on-transparent (the `_Branco` / "Blanco" variant), recolor it before deploying so it doesn’t disappear on white:

```python
import pathlib
p = pathlib.Path('force-app/main/default/staticresources/AscendumLogo.svg')
s = p.read_text()
s = s.replace('fill="white"', 'fill="#003846"')   # brand blue, visible on white header
s = s.replace('fill="#FFFFFF"', 'fill="#003846"')
# also paths with no fill attribute can be re-targeted via re.sub on '<path[^>]*>'
p.write_text(s)
```

Verify only the desired colors remain: `grep -oE 'fill="[^"]*"' AscendumLogo.svg | sort -u` should print just the new brand color (and `none` for stroke containers).

**Hero / banner image override (MANDATORY for client demos — CSP Trusted URL works here).** In `sfdc_cms__view/home/content.json` find **every** `"imageInfo": "{...\"imageInfoV1\":{\"url\":\"...\",...}}"` whose inner `url` is a stock template, SDO, or Cirrus image. The Cursor template ships `home-banner-2.jpg` for the main hero and `homeBanner-right.png` for the right secondary panel; B2B Commerce Enhanced seeded bundles can also include SDO images such as `emergency-prep-b2b.jpg` with visible text like `Emergency Prep`. Rewrite every such inner `url` to the brand's full `https://…` asset and replace the placeholder `altText` and visible text with a meaningful brand/category description. The `<img>` rendered by the home banner is a regular DOM image, so the `CspTrustedSite` allow-list is enough — nothing else to do. **The `tablet/tablet.json` and `mobile/mobile.json` files do not carry their own `imageInfo`; they inherit from `content.json`, so no extra edits there.** Always sanity-check the new URLs return `HTTP/2 200` with `content-type: image/*` before deploying:

```text
curl -sI -A "Mozilla/5.0" "<image_url>" | head -5
```

> **Skipping this step is the #1 way to ship a "branded" demo that still looks like the Cursor/SDO template.** Colors, copy and logo only get you ~70% of the way; home banners and seeded category tiles dominate the page. Always search the full home JSON for `assets/images/`, `sfdc-ckz-b2b.s3.amazonaws.com/SDO/Commerce_Images`, `Emergency Prep`, `Discover Nature`, `Solar`, `Energy`, `Wind`, `Battery`, and `Cirrus`.

**Parallax / custom component cleanup.** Seeded B2B bundles can include custom components such as `c:parallaxcmp` with stock attributes:

```json
{
  "definition": "c:parallaxcmp",
  "attributes": {
    "backgroundImage": "windEnergy",
    "mainTitle": "Discover Nature's Energy"
  }
}
```

At minimum replace `mainTitle` with a brand CSR/sustainability message. For example, for Dentaid use `Committed to Sustainable Oral Health`. If the component exposes only enumerated background images (for example `windEnergy`), changing the text is acceptable; do not leave the stock energy-themed title in a client demo.

Fallback only if the asset host cannot be whitelisted **and** static resource isn’t practical: embed an SVG via `content: url("data:image/svg+xml;base64,<BASE64>")` — but this only works for elements outside shadow DOM.

**Buyer-group-specific banner variants (validated with B2B Commerce LWR).** When Standard and VIP/Silver buyer groups need different home experiences, create the variants directly in `sfdc_cms__view/home/content.json` and deploy the bundle. This mirrors Experience Builder's **User > Commerce > Buyer Groups contains ...** rule.

Implementation pattern:

1. Deep-copy the target component tree (usually the first `dxp_content_layout:banner` on the home page).
2. Generate fresh UUIDs for every copied `id` so the cloned banner is a distinct component tree.
3. Change the cloned banner's `imageInfo`, text blocks, button labels, and colors for the VIP/Silver experience.
4. Add `contentOperations.operations` rules:

```json
{
  "targetId": "<VIP_BANNER_COMPONENT_ID>",
  "isHiddenOnOperationSuccess": false,
  "isActive": true,
  "rule": {
    "name": "<UUID>",
    "description": "Show VIP banner for VIP Customers Buyer Group",
    "criteriaType": "AllCriteriaMatch",
    "expressionCriteria": [{
      "resource": "User.Commerce.BuyerGroups",
      "operator": "Contains",
      "value": "<VIP_OR_SILVER_BUYER_GROUP_NAME>",
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
    "description": "Hide default banner for VIP Customers Buyer Group",
    "criteriaType": "AllCriteriaMatch",
    "expressionCriteria": [{
      "resource": "User.Commerce.BuyerGroups",
      "operator": "Contains",
      "value": "<VIP_OR_SILVER_BUYER_GROUP_NAME>",
      "criterionNumber": 1
    }]
  }
}
```

Deploy and publish after editing:

```text
sf project deploy start --target-org <alias> --source-dir force-app/main/default/digitalExperiences/site/<SITE_BUNDLE> --wait 20
sf community publish --name '<Site Name>' --target-org <alias>
```

Validation: log in with one buyer account that belongs only to the VIP/Silver group and one buyer account that belongs only to the Standard group. Accounts in both groups will match the VIP/Silver rule and see the VIP/Silver variant.

**Important:** do not use `User.AccountId`, `User.ProfileId`, `User.UserType`, or custom `User`/`Contact` fields for Commerce buyer-group visibility. B2B Commerce LWR deploy validation rejects those expressions with `Enter a valid expression`. The working metadata path is exactly `User.Commerce.BuyerGroups` + `Contains` + the buyer group name.

**Brand-color override pattern** (Ascendum example: `#003846` brand blue, `#E31D1A` accent red):

```css
:root {
    --dxp-g-brand: #003846;
    --dxp-g-brand-1: #002D38;
    --dxp-g-brand-2: #00222A;
    --dxp-g-brand-contrast: #FFFFFF;
    --dxp-g-link: #E31D1A;
    --dxp-g-link-hover: #B61715;
}
.slds-button_brand { background-color: #E31D1A !important; border-color: #E31D1A !important; }
h1, h2, h3 { color: #003846 !important; }
```

Update the matching color tokens **inside the brandingSet `values` object too** (`PrimaryAccentColor`, `_PrimaryAccentColor1..3`, `TextColor`, `_TextColor1..3`, `LinkColor`, `_LinkColor1`) so Builder previews and email theming match the rendered site.

After deploy, hard-refresh the storefront in an incognito window — the LWR runtime caches CSS aggressively per build version.

## Phase G — Org-level pricing engine + payment gateway sanity check

Two org-wide settings that must be in place before the storefront can render prices and accept payment. The pricing engine setting is **the most common cause of `Price Unavailable` in the storefront** even when every catalog/pricebook/buyer-group row is correct.

### G.1 — Salesforce Pricing engine (Recipe + Procedure + Sync)

**Setting in the UI (always works, use as fallback):** `Setup → Salesforce Pricing Setup` →

| Field | Value (SDO default) |
|---|---|
| Pricing Recipe | **`CommerceDefaultRecipe`** |
| Pricing Procedure | **`Cirrus - Commerce Default Pricing Procedure`** |

After picking both values, click **Sync**. Without this, the storefront returns **`Price Unavailable`** for every product even though `WebStorePricebook`, `BuyerGroupPricebook`, `PricebookEntry` and `CommerceEntitlementBuyerGroup` are all valid (it was the root cause of the parked §11.1 issue).

**Try CLI automation first** — the agent attempts these in order before resorting to the UI step:

```text
# 1. Discover whether the pricing-engine objects are exposed in this org's Data API
sf data query -q "SELECT Id, Name, IsActive FROM PricingProcedure WHERE Name LIKE 'Cirrus%' OR Name LIKE '%Commerce%'" -o <alias>
sf data query -q "SELECT Id, DeveloperName, MasterLabel FROM PricingRecipe"                                              -o <alias>

# 2. Check whether Commerce stores configuration via CommerceConfigRelatedRecord (same object used for SelfRegistration)
sf data query -q "SELECT Id, ConfigUseCase, ConfigKey, ConfigReferenceId FROM CommerceConfigRelatedRecord WHERE ConfigUseCase = 'Pricing' OR ConfigKey LIKE '%Recipe%' OR ConfigKey LIKE '%Procedure%'" -o <alias>
```

**If the discovery queries return rows** (objects are API-exposed), set the recipe/procedure via:

```text
# Pattern that works in SDOs that expose CommerceConfigRelatedRecord for Pricing.
# Substitute the IDs from the discovery queries.
sf data create record -s CommerceConfigRelatedRecord -o <alias> -v "ContextId=<WEBSTORE_ID> ConfigUseCase=Pricing ConfigKey=PricingRecipe    ConfigReferenceId=<COMMERCEDEFAULTRECIPE_ID>"
sf data create record -s CommerceConfigRelatedRecord -o <alias> -v "ContextId=<WEBSTORE_ID> ConfigUseCase=Pricing ConfigKey=PricingProcedure ConfigReferenceId=<CIRRUS_PROCEDURE_ID>"
```

The **Sync** action triggers procedure compilation; if no Apex/Connect REST equivalent is exposed in this org, the agent **must** stop and ask the user to click **Sync** in `Setup → Salesforce Pricing Setup`. Document the outcome: if neither query returns and the only working path is UI, capture that in `USER_MANUAL_STEPS.md` so the next bootstrap doesn't repeat the discovery loop.

**Verify the engine is wired** by hitting the buyer-side pricing API after Sync:

```text
sf api request rest "/services/data/v62.0/commerce/webstores/<WEBSTORE_ID>/pricing/products/<SAMPLE_PRODUCT_ID>" --target-org <alias>
```

A `200` with `unitPrice` populated means the engine is live; `PRICE_NOT_FOUND` means Recipe/Procedure are still wrong or Sync wasn't run.

### G.2 — Payment gateway

- Confirm a **`PaymentGateway`** row exists (**`SalesforcePG`**, **Active**) before relying on **`StoreIntegratedService`** (**Phase C**, subsection **StoreIntegratedService**).
- **`WebStore`** has no reliable **`PaymentGatewayId`** field — the store links to the gateway via **`StoreIntegratedService`** where **`ServiceProviderType=Payment`** and **`Integration=<PaymentGateway.Id>`**.

```text
sf data query -q "SELECT Id, PaymentGatewayName, Status FROM PaymentGateway WHERE PaymentGatewayName = 'SalesforcePG'" -o <alias>
```

## Phase H — Search index

After catalog/buyer/shipping changes:

```text
sf commerce search start --store-name '<WebStore Name>' --targetusername <alias>
```

**Commerce “automatic” / incremental index updates** (UI wording) are not exposed as a single, stable field on **`WebStore`** in all orgs. If the org does not offer a Connect/Metadata toggle the agent can set safely, complete that step in **Commerce workspace for the store → Search** (enable scheduled or incremental updates per product help for your release). Do not invent boolean fields on **`WebStore`** without verifying describe output in the target org.

## Phase Z — Self-validation checklist (mandatory final gate)

**Hard rule for the agent:** before handing the storefront back to the user (URL + buyer credentials) or kicking off Phase I / the catalog generator, the agent **must** run every check below itself and print a pass/fail table. The phase is the contract that proves the previous phases actually wired the store correctly — symptoms like "guests can't see prices", "mega-menu is empty", "self-registered users have no catalog access" all surface here as concrete query results, not as user complaints later.

If any check fails, **do not** continue to Phase I. Loop back to the phase that owns the failing check, fix it, and re-run Phase Z. Only when every row prints **PASS** is the storefront declared ready.

The agent runs the checks in this exact order, capturing each query result and asserting the expected shape **before** moving on:

### Z.1 — Phase A: Site + WebStore + guest enablement

```text
# Z.1.a — Network is Live with internal-login + self-reg + embedded-login enabled
sf data query -q "SELECT Id, Status, OptionsAllowInternalUserLogin, OptionsSelfRegistrationEnabled, OptionsEmbeddedLoginEnabled, SelfRegProfileId FROM Network WHERE Id = '<NETWORK_ID>'" -o <alias>
# Expect: Status=Live, all three Options*=true, SelfRegProfileId NOT null

# Z.1.b — WebStore guest browsing/cart/checkout flags
sf data query -q "SELECT OptionsGuestBrowsingEnabled, OptionsGuestCartEnabled, OptionsGuestCheckoutEnabled, OptionsPreserveGuestCartEnabled FROM WebStore WHERE Id = '<WEBSTORE_ID>'" -o <alias>
# Expect: every flag = true

# Z.1.c — Site authenticationType (deployed bundle JSON)
grep '"authenticationType"' force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/sfdc_cms__site/<SITE_BUNDLE>/content.json
# Expect: "authenticationType" : "AUTHENTICATED_WITH_PUBLIC_ACCESS_ENABLED"

# Z.1.d — NavigationMenu Storefront Categories publicly available
grep -B2 -A4 '<target>StorefrontCategories</target>' force-app/main/default/navigationMenus/*.navigationMenu-meta.xml
# Expect: <publiclyAvailable>true</publiclyAvailable> on the StorefrontCategories item
```

### Z.2 — Phase B: Catalog scaffolding + "All Products" category + 3-pricebook layout

```text
# Z.2.a — Catalog wired to WebStore
sf data query -q "SELECT Id, ProductCatalogId, SalesStoreId FROM WebStoreCatalog WHERE SalesStoreId = '<WEBSTORE_ID>'" -o <alias>
# Expect: exactly 1 row, ProductCatalogId = <CATALOG_ID>

# Z.2.b — Mandatory "All Products" category exists under the catalog
sf data query -q "SELECT Id, Name FROM ProductCategory WHERE CatalogId = '<CATALOG_ID>' AND Name = 'All Products'" -o <alias>
# Expect: exactly 1 row

# Z.2.c — Three custom pricebooks active and wired to the WebStore
sf data query -q "SELECT WebStorePricebook.Pricebook2.Name, WebStorePricebook.IsActive FROM WebStorePricebook WHERE WebStoreId = '<WEBSTORE_ID>' ORDER BY WebStorePricebook.Pricebook2.Name" -o <alias>
# Expect: 3 rows: '<Site> List Price Book', '<Site> Sale Price Book', '<Site> VIP Price Book', all IsActive=true

# Z.2.d — Strikethrough pricebook set on WebStore
sf data query -q "SELECT StrikethroughPricebookId FROM WebStore WHERE Id = '<WEBSTORE_ID>'" -o <alias>
# Expect: equal to <LIST_PB_ID>

# Z.2.e — All 3 custom pricebooks in the configured currency (multi-currency orgs only)
sf data query -q "SELECT Name, CurrencyIsoCode FROM Pricebook2 WHERE Id IN ('<LIST_PB_ID>','<SALE_PB_ID>','<VIP_PB_ID>')" -o <alias>
# Expect: every row CurrencyIsoCode = <CURRENCY> from Phase 0 Q4
```

### Z.3 — Phase C/D: StoreIntegratedService + Shipping

```text
# Z.3.a — All six StoreIntegratedService rows for parity with SDO
sf data query -q "SELECT ServiceProviderType, Integration FROM StoreIntegratedService WHERE StoreId = '<WEBSTORE_ID>' ORDER BY ServiceProviderType" -o <alias>
# Expect: 6 rows — Inventory, Payment, Price, Promotions, Shipment, Tax (Payment.Integration = PaymentGateway.Id)

# Z.3.b — At least one ShippingConfigurationSet for the WebStore
sf data query -q "SELECT Id, Name, IsDefault FROM ShippingConfigurationSet WHERE TargetRecordId = '<WEBSTORE_ID>'" -o <alias>
# Expect: >= 1 row, exactly one with IsDefault=true
```

### Z.4 — Phase E: Buyer groups + accounts + GUEST profile linkage

```text
# Z.4.a — Two buyer groups attached to the WebStore
sf data query -q "SELECT BuyerGroup.Name FROM WebStoreBuyerGroup WHERE WebStoreId = '<WEBSTORE_ID>' ORDER BY BuyerGroup.Name" -o <alias>
# Expect: 2 rows — Standard Customers + VIP Customers

# Z.4.b — Both buyer groups carry the Phase B entitlement policy
sf data query -q "SELECT BuyerGroup.Name, Policy.Name FROM CommerceEntitlementBuyerGroup WHERE BuyerGroupId IN ('<STD_BG_ID>','<VIP_BG_ID>')" -o <alias>
# Expect: 2 rows pointing at the same CommerceEntitlementPolicy

# Z.4.c — Pricebook priorities (Sale/VIP=1, List=2 for strikethrough)
sf data query -q "SELECT BuyerGroup.Name, Pricebook2.Name, Priority, IsActive FROM BuyerGroupPricebook WHERE BuyerGroupId IN ('<STD_BG_ID>','<VIP_BG_ID>') ORDER BY BuyerGroup.Name, Priority" -o <alias>
# Expect: 4 rows total — 2 per group, Priority=1 + Priority=2, all IsActive=true

# Z.4.d — Both buyer accounts active and Account.IsBuyer flipped
sf data query -q "SELECT Account.Name, Account.IsBuyer, BuyerStatus, IsActive FROM BuyerAccount WHERE BuyerId IN ('<STD_ACCOUNT_ID>','<VIP_ACCOUNT_ID>')" -o <alias>
# Expect: 2 rows, IsBuyer=true AND IsActive=true AND BuyerStatus=Active

# Z.4.e — CRITICAL — site Guest Buyer Profile is a member of the Standard buyer group
sf data query -q "SELECT GuestBuyerProfileId FROM WebStore WHERE Id = '<WEBSTORE_ID>'" -o <alias>
# capture <SITE_GUEST_BUYER_PROFILE_ID>
sf data query -q "SELECT BuyerGroup.Name FROM BuyerGroupMember WHERE BuyerId = '<SITE_GUEST_BUYER_PROFILE_ID>'" -o <alias>
# Expect: >= 1 row, BuyerGroup.Name = '<Site Name> Standard Customers'
# This is the gate that prevents the "guest sees catalog but every product shows Price Unavailable" failure.
```

### Z.5 — Phase F: Network member groups + buyer Users + Site Guest User PSAs

```text
# Z.5.a — Buyer profile + 3 Commerce permission sets are NetworkMemberGroup rows
sf data query -q "SELECT Parent.Name, ParentId FROM NetworkMemberGroup WHERE NetworkId = '<NETWORK_ID>' ORDER BY Parent.Name" -o <alias>
# Expect: at minimum 4 rows — B2B Reordering Portal Buyer Profile, B2B_Commerce_Guest_Browser_Access, B2BCommerce_Community_Access, D2C_Commerce_Guest_User_Access

# Z.5.b — Standard + VIP buyer Users active with 3 PSAs each
sf data query -q "SELECT Username, IsActive, ContactId FROM User WHERE ContactId IN ('<STD_CONTACT_ID>','<VIP_CONTACT_ID>')" -o <alias>
# Expect: 2 rows, IsActive=true
sf data query -q "SELECT Assignee.Username, PermissionSet.Name FROM PermissionSetAssignment WHERE AssigneeId IN (SELECT Id FROM User WHERE ContactId IN ('<STD_CONTACT_ID>','<VIP_CONTACT_ID>')) ORDER BY Assignee.Username, PermissionSet.Name" -o <alias>
# Expect: 6 rows — B2BBuyer + B2BCommerce_Community_Access + B2B_Commerce_Customer_Community_Plus_Access for each user

# Z.5.c — Site Guest User PSAs for guest browsing
sf data query -q "SELECT Id FROM User WHERE Profile.Name = '<Site Name> Profile' LIMIT 1" -o <alias>
# capture <SITE_GUEST_USER_ID>
sf data query -q "SELECT PermissionSet.Name FROM PermissionSetAssignment WHERE AssigneeId = '<SITE_GUEST_USER_ID>' ORDER BY PermissionSet.Name" -o <alias>
# Expect: at minimum B2B_Commerce_Guest_Browser_Access + D2C_Commerce_Guest_User_Access

# Z.5.d — Buyer + Guest user currency matches Phase 0 Q4 (multi-currency orgs only)
sf data query -q "SELECT Username, DefaultCurrencyIsoCode, LanguageLocaleKey, LocaleSidKey FROM User WHERE Id IN ('<STD_BUYER_USER_ID>','<VIP_BUYER_USER_ID>','<SITE_GUEST_USER_ID>')" -o <alias>
# Expect: every row DefaultCurrencyIsoCode = <CURRENCY>. This is the #1 silent cause
# of 'Price Unavailable' on the storefront — the pricing engine filters
# PricebookEntry by User.DefaultCurrencyIsoCode, and new users inherit the org's
# corporate currency regardless of WebStore configuration.
```

### Z.6 — Phase F2: Branding deployed correctly

```text
# Z.6.a — All four brandingSets carry the brand colors (NOT the template defaults)
for set in B2B_Commerce B2B_Footer B2B_Home_Banner B2B_Right_Panel; do
  echo "== $set =="
  grep -E '"(PrimaryAccentColor|LinkColor|TextColor)"' \
    force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/sfdc_cms__brandingSet/$set/content.json
done
# Expect: brand hex codes (NOT the Cursor #1C1C1C / template defaults) in all four sets

# Z.6.b — SiteLogo points at a deployed StaticResource
grep -E '"(SiteLogo|_SiteLogoUrl)"' force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/sfdc_cms__brandingSet/B2B_Commerce/content.json
# Expect: "/sfsites/c/resource/<ResourceName>" and "url(/sfsites/c/resource/<ResourceName>)" — never "assets/images/<stock>", never "/resource/<ResourceName>", and never an external URL

# Z.6.c — Both home banner imageInfo URLs replaced (NOT the stock template URLs)
grep -E 'home-banner-2|homeBanner-right' force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/sfdc_cms__view/home/content.json
# Expect: NO matches (both stock URLs replaced with brand-relevant external URLs)

# Z.6.d — geoBotsAllowed stripped from the deployed bundle
grep -r 'geoBotsAllowed' force-app/main/default/digitalExperiences/site/<SITE_BUNDLE>/ || echo "OK — none found"
# Expect: "OK — none found"
```

### Z.7 — Phase G: Pricing engine sanity check (live API call)

```text
# Hit the buyer-side pricing API as the guest — most reliable signal that the
# whole stack (CommerceEntitlementBuyerGroup + BuyerGroupPricebook + Pricing
# engine + GuestBuyerProfile membership) is wired. Note: this only works once
# at least one product has a PricebookEntry. Skip Z.7 entirely when running
# the storefront-only flow before Phase I.
sf data query -q "SELECT Id FROM Product2 WHERE Id IN (SELECT ProductId FROM CommerceEntitlementProduct WHERE PolicyId = '<POLICY_ID>') LIMIT 1" -o <alias>
# If 0 rows → SKIP Z.7 (no products to price); proceed to handover.
# If >= 1 row → capture <SAMPLE_PRODUCT_ID> and:
sf api request rest "/services/data/v62.0/commerce/webstores/<WEBSTORE_ID>/pricing/products/<SAMPLE_PRODUCT_ID>" --target-org <alias>
# Expect: HTTP 200 with unitPrice populated. PRICE_NOT_FOUND or HTTP 4xx => Phase G.1 not wired or Z.4.e not done.
```

### Z.8 — Print pass/fail table and decide

After running Z.1 through Z.7, the agent prints a single summary table to the user:

```text
| Check     | Phase   | Status | Notes                                        |
|-----------|---------|--------|----------------------------------------------|
| Z.1.a     | A       | PASS   | Network Live, login flags ok                 |
| Z.1.b     | A       | PASS   | WebStore guest flags ok                      |
| Z.1.c     | A       | PASS   | authenticationType correct                   |
| Z.1.d     | A       | PASS   | StorefrontCategories publiclyAvailable=true  |
| Z.2.a     | B       | PASS   | Catalog wired                                |
| Z.2.b     | B       | PASS   | All Products category present                |
| Z.2.c     | B       | PASS   | 3 pricebooks active                          |
| Z.2.d     | B       | PASS   | StrikethroughPricebookId set                 |
| Z.3.a     | C       | PASS   | 6 StoreIntegratedService rows                |
| Z.3.b     | D       | PASS   | Default ShippingConfigurationSet exists      |
| Z.4.a-d   | E       | PASS   | Buyer groups + accounts + pricebook tiers    |
| Z.4.e     | E       | PASS   | Guest profile linked to Standard group       |
| Z.5.a-c   | F       | PASS   | Network members + buyer users + guest PSAs   |
| Z.6.a-d   | F2      | PASS   | Branding deployed (colors + logo + banners)  |
| Z.7       | G       | SKIP   | No products yet — covered by Phase I         |
```

If **every** non-skipped row is **PASS**, the agent emits the handover block:

```text
✅ Storefront ready.
   • URL: https://<MyDomain>/<UrlPathPrefix>/s/
   • Standard buyer: standard.buyer@<suffix>.demo / Buyer!Demo123
   • VIP buyer:      vip.buyer@<suffix>.demo / Buyer!Demo123
   • Guest:          (no login required)

Next: would you like to load the product catalog now (Phase I / sf-b2b-catalog-generator), or hand the storefront back as-is?
```

If **any** row is **FAIL**, the agent reports which phase owns the failing check, fixes it, and re-runs Phase Z from the start. **Never hand the storefront back with FAIL rows still present** — every check in Phase Z is a parked failure mode that has shipped to a user before, and the whole phase exists to catch them automatically.

## Phase I — Product catalog content (optional, deferred — last step)

**Hard rule for the agent:** Phase I is **never run automatically** as part of the initial bootstrap. After Phases A → H finish, the agent hands the storefront back to the user (URL + buyer credentials) and **asks** whether to proceed with Phase I or to manage the catalog separately. The user normally wants to revisit catalog tooling on its own track — do not push Phase I unprompted.

When the user does ask for Phase I, the inputs come from the JSON saved during Phase 0 (`branding-work/<slug>/brand.json`) and the IDs from `branding-work/<slug>/store-ids.json` (Phase B). Phase I writes to:

- `ProductCategory` (one per category, plus the mandatory `All Products` top-level category).
- `Product2` (one per SKU).
- `PricebookEntry` (one per SKU per pricebook — Standard, Sale, VIP).
- `ProductCategoryProduct` (link product → category, plus the All Products link).
- `CommerceEntitlementProduct` (link product → policy from Phase B).
- Optional: `ProductMedia` after CMS image upload.

### One-shot bootstrap commands (ordered)

```text
# 1. ProductCategory rows (one per category) — all under the Phase B catalog
sf data create record -s ProductCategory -v "Name='<Category Name>' CatalogId=<CATALOG_ID> Description='...'" -o <alias>
# ... repeat for every category

# 2. Product2 rows — Family for grouping, ProductCode + StockKeepingUnit identical
sf data create record -s Product2 -v "Name='<Product Name>' ProductCode='<SKU>' StockKeepingUnit='<SKU>' Family='<Family>' IsActive=true Description='...'" -o <alias>
# ... repeat for every product

# 3. PricebookEntry — org Standard PB FIRST (Salesforce platform requirement: a product
#    cannot have a PricebookEntry on any custom Pricebook2 until it has one on the org
#    Standard Pricebook2). Then List (custom — the WebStore-facing strikethrough), then
#    Sale, then VIP. UnitPrice on Standard and List should match the list price.
#    <CURRENCY> must match Phase 0 Q4 AND the CurrencyIsoCode stamped on each Pricebook2
#    in Phase B. On single-currency orgs, drop `CurrencyIsoCode=<CURRENCY>` from every command.
sf data create record -s PricebookEntry -v "Pricebook2Id=<STANDARD_PB_ID> Product2Id=<PROD_ID> UnitPrice=<list_price>  IsActive=true UseStandardPrice=false CurrencyIsoCode=<CURRENCY>" -o <alias>
sf data create record -s PricebookEntry -v "Pricebook2Id=<LIST_PB_ID>     Product2Id=<PROD_ID> UnitPrice=<list_price>  IsActive=true UseStandardPrice=false CurrencyIsoCode=<CURRENCY>" -o <alias>
sf data create record -s PricebookEntry -v "Pricebook2Id=<SALE_PB_ID>     Product2Id=<PROD_ID> UnitPrice=<sale_price>  IsActive=true UseStandardPrice=false CurrencyIsoCode=<CURRENCY>" -o <alias>
sf data create record -s PricebookEntry -v "Pricebook2Id=<VIP_PB_ID>      Product2Id=<PROD_ID> UnitPrice=<vip_price>   IsActive=true UseStandardPrice=false CurrencyIsoCode=<CURRENCY>" -o <alias>

# 4. Link product → category
sf data create record -s ProductCategoryProduct -v "ProductId=<PROD_ID> ProductCategoryId=<CAT_ID>" -o <alias>

# 5. ALWAYS create a top-level "All Products" category and link every product to it
sf data create record -s ProductCategory        -v "Name='All Products' CatalogId=<CATALOG_ID> Description='Browse the full catalog.'" -o <alias>
sf data create record -s ProductCategoryProduct -v "ProductId=<PROD_ID> ProductCategoryId=<ALL_PRODUCTS_CAT_ID>"                       -o <alias>   # repeat for every product

# 6. Link policy → product (catalog-level entitlement is not exposed; we list products)
sf data create record -s CommerceEntitlementProduct -v "PolicyId=<POLICY_ID> ProductId=<PROD_ID>" -o <alias>
```

### Always finish with reindex + republish

```text
sf commerce search start --store-name '<Site Name>' --targetusername <alias>
sf community publish     --name        '<Site Name>' --target-org    <alias>
```

### Product images — known limitation

`Product2.DisplayUrl` is **not** consumed by the LWR product card. Images must live as **`ManagedContent` (`ContentTypeFullyQualifiedName='sfdc_cms__image'`)** in the CMS workspace linked to the site, then be linked to the product via **`ProductMedia.ElectronicMediaId`**.

The Connect REST endpoint **`POST /connect/cms/contents`** in this org accepts the JSON shape

```json
{
  "contentSpaceOrFolderId": "<SPACE_ID>",
  "contentType": "sfdc_cms__image",
  "title": "...",
  "contentBody": { "sfdc_cms:media": { "source": { "type": "ContentReference", "ref": "contentData" } } }
}
```

but rejects every multipart envelope tried from `sf api request rest` and from `curl -F` (errors flip between *"A request body is required"* and *"INVALID_API_INPUT"* depending on the part-name combination). The **Apex** path also fails to compile in this org because the modern `ConnectApi.CmsContent` / `ConnectApi.ManagedContentNodeValueInput` types are not exposed. **Recommended workflow today:** upload the images by hand in the CMS workspace UI, then bulk-create `ProductMedia` rows from the CLI — full step-by-step in **`USER_MANUAL_STEPS.md` §11.4**.

### Reference catalog (Ascendum demo, kept for parity)

| SKU | Name | Family | Category | List price (EUR) |
|---|---|---|---|---|
| VOL-L150H | Volvo L150H Wheel Loader | Construction | Construction Equipment | 185 000 |
| VOL-A30G | Volvo A30G Articulated Hauler | Construction | Construction Equipment | 245 000 |
| VOL-EC260E | Volvo EC260E Crawler Excavator | Construction | Construction Equipment | 198 500 |
| PON-BUFFALO | Ponsse Buffalo Forwarder | Construction | Construction Equipment | 420 000 |
| VOL-FH16-750 | Volvo FH16 750 Tractor Unit | Trucks | Trucks | 165 000 |
| REN-THIGH-520 | Renault Trucks T High 520 | Trucks | Trucks | 128 000 |
| VOL-FMX460 | Volvo FMX 460 8x4 Tipper | Trucks | Trucks | 142 500 |
| VAL-Q305 | Valtra Q305 Tractor | Agro | Agro Equipment | 189 000 |
| FEN-942-VARIO | Fendt 942 Vario | Agro | Agro Equipment | 298 000 |
| MF-8740S | Massey Ferguson 8740 S | Agro | Agro Equipment | 172 000 |
| PRT-FILTERKIT | Volvo Genuine Service Filter Kit | Parts | Parts & Attachments | 485 |
| PRT-QC-BUCKET | Quick-Coupler Bucket 1.6 m³ | Parts | Parts & Attachments | 4 250 |
| SVC-CAREPLUS | Ascendum Care+ Maintenance Plan (12 months) | Service | Service Plans | 3 950 |
| SVC-UPTIME | Ascendum Uptime Telematics Subscription (1 year) | Service | Service Plans | 1 290 |

Full JSON definition lives in `/Users/dsiguenza/Documents/SFCore/cursorTesting/catalog-work/catalog.json` (resolved IDs in `category-ids.json`, `product-ids.json`, `pbe-ids.json`). Reuse as a template for new client/industry catalogs in Phase I.

## Troubleshooting — guest redirect, self-registration, “Login as” buyer, prices, images

| Symptom | Likely cause | Fix (verify in target org) |
|--------|----------------|----------------------------|
| Anonymous users always sent to login | LWR site still on **`AUTHENTICATED`** (see **`sfdc_cms__site` `authenticationType`**), **public access** not enabled, **`OptionsEmbeddedLoginEnabled=false`**, or pages login-only | **Preferred:** deploy **`AUTHENTICATED_WITH_PUBLIC_ACCESS_ENABLED`** on the **`DigitalExperienceBundle`** site JSON (see Phase A). Set **`Network.OptionsEmbeddedLoginEnabled=true`**; confirm **`WebStore.OptionsGuestBrowsingEnabled=true`**. UI fallback: Builder **Settings → General** public access + **Publish**. |
| Guests reach the storefront but mega menu / categories / catalog are empty | **Site Guest User** is missing the Commerce guest permission sets (FLS on Product / ProductCategory / Pricebook for the guest profile is locked down) | Assign these permission sets directly to the site **Guest User** (mirror SDO): **`B2B_Commerce_Guest_Browser_Access`**, **`D2C_Commerce_Guest_User_Access`**, and (in SDO) **`SDO_B2B_Commerce_Guest_Access`**. Use **`PermissionSetAssignment`** with **`AssigneeId = <Site Guest User Id>`**. |
| Guests see Home but not category links in the mega menu | **`NavigationMenu`** has **`StorefrontCategories`** with **`publiclyAvailable`** false | **Preferred:** deploy **`NavigationMenu`** with **`publiclyAvailable` true**. UI fallback: Builder **Navigation** → **Storefront Categories** → **Publicly available** → **Publish**. |
| Storefront shows products **without prices** (preview or buyer login) | **`WebStorePricebook.IsActive`** and/or **`BuyerGroupPricebook.IsActive`** = **false** for the new store; entitlement is OK but the buyer-side price tier is not active | Update each row to **`IsActive=true`** via **`sf data update record -s WebStorePricebook -i <ID> -v "IsActive=true"`** and **`sf data update record -s BuyerGroupPricebook -i <ID> -v "IsActive=true"`**. Then re-run **`sf commerce search start`** and **`sf community publish`**. |
| Logged-in buyer (and preview) shows **`Price Unavailable`** even though `WebStorePricebook` and `BuyerGroupPricebook` are active and `PricebookEntry` exists | **Three common root causes — check in this order.** (1) Most common: the **Salesforce Pricing engine is not wired** for the org (`Setup → Salesforce Pricing Setup` has no Recipe/Procedure selected or Sync was never clicked). (2) **User/Pricebook currency mismatch** — the buyer User's `DefaultCurrencyIsoCode` does not match the `PricebookEntry.CurrencyIsoCode`; the pricing engine filters entries by the User's currency. New Users (including the site Guest User) inherit the org's **corporate currency** at creation regardless of WebStore configuration, so if your storefront is EUR and the corporate currency is USD, every buyer silently gets `Price Unavailable`. (3) A `BuyerGroup` with multiple active `BuyerGroupPricebook` rows where every row has `Priority = null`. | **First**, complete **Phase G.1** — set Pricing Recipe = `CommerceDefaultRecipe`, Pricing Procedure = `Cirrus - Commerce Default Pricing Procedure`, click **Sync**. **Second**, verify every buyer-facing user: `sf data query -q "SELECT Username, DefaultCurrencyIsoCode FROM User WHERE Id IN ('<STD_BUYER_ID>','<VIP_BUYER_ID>','<SITE_GUEST_USER_ID>')"` and update any mismatch with `sf data update record -s User -i <USER_ID> -v "DefaultCurrencyIsoCode=<CURRENCY>"`. **Third**, if still broken, set unique `BuyerGroupPricebook.Priority` per group. Re-run `sf commerce search start` and `sf community publish`. |
| Storefront shows products **without images** | Product images live in a CMS workspace (e.g. **`All Commerce - Enhanced`**) that is **not linked** to the new site’s channels (the site has its own **Community** + **PublicUnauthenticated** **`ManagedContentChannel`** rows). Even after linking, content must be **republished** to the new channels. | Link channels via **`PATCH /services/data/v62.0/connect/cms/spaces/<SPACE_ID>/channels`** with body **`{"spaceChannels":[{"channelId":"<COMM_CH>","operation":"Add"},{"channelId":"<PUB_CH>","operation":"Add"}]}`**. **Then in the UI**: open the workspace → **Content** → **Select all** → **Publish** (the Connect REST `POST /connect/cms/contents/publish` returns a generic *Your content wasn’t published. Try again.* error against `All Commerce - Enhanced`, regardless of payload). |
| Cannot complete self-registration | Missing **`SelfRegProfileId`**, validation on Account/Contact, duplicate username, or Commerce config | Set **`SelfRegProfileId`** to **`B2B Reordering Portal Buyer Profile`** (or equivalent). Confirm **`CommerceConfigRelatedRecord`** rows for **`SelfRegistration` + DefaultBuyerGroup** and **AccountRecordType**. Check object rules and duplicate email/username. |
| Admin **Login as** user but storefront shows not logged in | Wrong URL, session not on community host, or tab opened before impersonation | From **Setup → user → Login**, start **Login as**, then open the **Experience** base URL in the **same browser session**: `https://<MyDomain>/` + **`Network.UrlPathPrefix`** + `/` (e.g. `…/cursorcommercedemovforcesite/s/`). Use the **community user’s** username domain (e.g. `@…my.site.com` suffix) — not the internal `…force.com` admin URL for the storefront test. |
| Internal admin **Login as** redirects to login / session does not stick on the storefront | **`Network.OptionsAllowInternalUserLogin = false`** blocks the internal session from being recognised by the Experience site | `sf data update record -s Network -i <NETWORK_ID> -v "OptionsAllowInternalUserLogin=true"` then `sf community publish`. |
| Authenticated buyer logs in but **header shows “Log In”** and **catalog is empty** | Self-registered / cloned buyer **User** has **no Commerce buyer permission sets assigned** (the profile is in **`NetworkMemberGroup`** so the user becomes a **`NetworkMember`**, but without the buyer PSAs the runtime treats the request as un-entitled and the LWR header falls back to the guest variant) | Assign these three permission sets to **every** buyer **User** (mirror SDO buyers like `James Wu`, `Lauren Bailey`): **`B2BBuyer`** (label *Buyer*), **`B2BCommerce_Community_Access`**, **`B2B_Commerce_Customer_Community_Plus_Access`**. Use **`PermissionSetAssignment`** with **`AssigneeId = <buyer User Id>`**. Also add them to your **self-registration handler** (Apex / Flow) so every newly-registered buyer gets them automatically. |

### CMS workspace ↔ site channel: discover and link via CLI

```text
sf data query -q "SELECT Id, AuthoredManagedContentSpaceId FROM ManagedContent WHERE Id IN (SELECT ElectronicMediaId FROM ProductMedia WHERE ProductId = '<SAMPLE_PRODUCT_ID>')" -o <alias>
sf data query -q "SELECT Id, Name, Type FROM ManagedContentChannel WHERE Name LIKE '<Site Name>%'" -o <alias>
sf api request rest "/services/data/v62.0/connect/cms/spaces/<SPACE_ID>/channels" --target-org <alias>
sf api request rest --method PATCH --body '{"spaceChannels":[{"channelId":"<COMM_CH>","operation":"Add"},{"channelId":"<PUB_CH>","operation":"Add"}]}' "/services/data/v62.0/connect/cms/spaces/<SPACE_ID>/channels" --target-org <alias>
```

After channels are added, **publish content from the UI** (CMS workspace → Content → Publish) — the matching Connect REST publish currently returns the generic error in this org.

## Verification checklist

- [ ] **`WebStore`**: tax policy, location, guest flags, price books, lifecycle match SDO intent.
- [ ] **`StoreIntegratedService`**: six rows for parity with SDO (**Payment** row uses **`PaymentGateway.Id`** for **SalesforcePG**; **Tax** = **`Tax__COMPUTE_TAXES`**).
- [ ] **`ShippingConfigurationSet`** exists for **`TargetRecordId = WebStore.Id`** with default + special profiles as needed.
- [ ] **`ShippingConfigSetProduct`** for special products (if SDO uses them).
- [ ] **Default 3-pricebook layout**: `<Site> List Price Book` + `<Site> Sale Price Book` + `<Site> VIP Price Book` (all three **custom** — the org `Standard Price Book` is rejected by `WebStorePricebook` / `BuyerGroupPricebook`), all three present in **`WebStorePricebook`** with `IsActive=true`. **`WebStore.StrikethroughPricebookId`** points at the **List** pricebook.
- [ ] **Default buyer layout**: 2 `BuyerGroup` rows (**`Standard Customers`** + **`VIP Customers`**), 2 `Account` + `BuyerAccount` (`BuyerStatus=Active` AND **`IsActive=true`**, so `Account.IsBuyer` flipped to `true`), 2 `Contact`, 2 `BuyerGroupMember`, and 4 `BuyerGroupPricebook` rows with `Priority=1` on the offer pricebook (Sale or VIP) and `Priority=2` on the **List** pricebook for strikethrough.
- [ ] **Phase F automation complete**: 2 buyer `User` rows active (Standard + VIP) with **3 PSAs each** (`B2BBuyer`, `B2BCommerce_Community_Access`, `B2B_Commerce_Customer_Community_Plus_Access`), Site Guest User has `B2B_Commerce_Guest_Browser_Access` + `D2C_Commerce_Guest_User_Access`, `Network.OptionsAllowInternalUserLogin=true`, `SelfRegProfileId` set.
- [ ] **Phase G.1 — Salesforce Pricing engine** wired: `Setup → Salesforce Pricing Setup` shows **Pricing Recipe = `CommerceDefaultRecipe`**, **Pricing Procedure = `Cirrus - Commerce Default Pricing Procedure`**, and **Sync** has been clicked (or the agent confirmed the equivalent CLI path returned a `200`/active config). Without this the storefront returns **`Price Unavailable`** for every product.
- [ ] **`WebStoreBuyerGroup`** + **`BuyerGroupMember`** + **`BuyerAccount`** consistent.
- [ ] **`CommerceEntitlementBuyerGroup`** links each store buyer group to the correct **`CommerceEntitlementPolicy`** (otherwise **no prices**).
- [ ] For **guest** sessions: a **`Market`** **`BuyerGroup`** with **`BuyerGroupBuyerCriteria`** (locale) is on the store’s **`WebStoreBuyerGroup`** list; **AccountBased** groups alone are not enough for anonymous pricing.
- [ ] **Community user** can log in (`NetworkMember` present); guest can browse (**Builder public access** + **`WebStore.OptionsGuestBrowsingEnabled`** + **`Network.OptionsEmbeddedLoginEnabled`**); nav **Storefront Categories** is **Publicly available** where used.
- [ ] **Site Guest User** has the **Commerce guest permission sets** assigned (see Troubleshooting — empty mega menu).
- [ ] **Authenticated buyer Users** have **`B2BBuyer`**, **`B2BCommerce_Community_Access`**, **`B2B_Commerce_Customer_Community_Plus_Access`** assigned (otherwise login appears to “not stick” and catalog is hidden).
- [ ] **`Network.OptionsAllowInternalUserLogin=true`** when admins need **Login as** to land authenticated on the storefront.
- [ ] **`WebStorePricebook`** + **`BuyerGroupPricebook`** rows for the new store are **`IsActive=true`** (otherwise prices render blank).
- [ ] When a **`BuyerGroup`** has more than one active **`BuyerGroupPricebook`**, every row must have a unique **`Priority`** integer (`1` = highest). All-null priorities trigger **`Price Unavailable`** even with valid `PricebookEntry`.
- [ ] **CMS workspace** holding product images is linked to the new site’s **community** + **publicunauthenticated** channels and content has been **republished** to those channels (**only if Phase I has run** — base bootstrap may have an empty CMS workspace).
- [ ] **Checkout**: run a browser test—shipping methods/rates appear; payment step completes or surfaces gateway errors (then fix in Payments setup). **Skip this check until Phase I has loaded products** — empty stores cannot test checkout end-to-end.
- [ ] **`Network.SelfRegProfileId` set in correct order** (Phase A): activated `Status=Live` first, inserted `NetworkMemberGroup` for the buyer profile next, **then** updated `SelfRegProfileId`. Setting it before the profile is a member fails with *"You can only select profiles that are associated with the experience."*
- [ ] **`geoBotsAllowed` stripped** from `sfdc_cms__site/<SITE>/content.json` before deploying the `DigitalExperienceBundle` (Phase F2). Otherwise the deploy fails with *"You can't add the geoBotsAllowed property…"*
- [ ] **All four brandingSets updated** (Phase F2): color tokens applied to `B2B_Commerce`, `B2B_Footer`, `B2B_Home_Banner`, `B2B_Right_Panel`. The `SiteLogo` keys only live in `B2B_Commerce`.
- [ ] **Both homepage banner images replaced** (Phase F2): `imageInfo.url` in `sfdc_cms__view/home/content.json` rewritten away from the stock `assets/images/home-banner-2.jpg` (main hero) and `assets/images/homeBanner-right.png` (right panel) to brand-relevant external URLs whose host is in `CspTrustedSite` with `isApplicableToImgSrc=true`. Both URLs verified to return `HTTP/2 200` with `content-type: image/*`. Otherwise the homepage above-the-fold still looks like the Cursor template.
- [ ] **Mandatory "All Products" `ProductCategory`** (Phase B Step 1) exists under the Phase B catalog with the exact name `All Products`. The catalog generator (Phase I) attaches every imported product to this category in addition to its specific one — without it, freshly created stores ship with an empty mega-menu until at least one specific category exists.
- [ ] **Site Guest Buyer Profile is a member of the Standard buyer group** (Phase E step 9): `SELECT BuyerGroup.Name FROM BuyerGroupMember WHERE BuyerId = WebStore.GuestBuyerProfileId` returns the Standard group. Without this row, anonymous visitors reach the catalog menu (Phase A) but every product renders `Price Unavailable` because the guest has no buyer-group → pricebook chain.
- [ ] **Currency alignment (Phase 0 Q4 `<CURRENCY>`) across pricebooks AND users** (multi-currency orgs only): all three custom `Pricebook2` rows carry `CurrencyIsoCode = <CURRENCY>`, every `PricebookEntry` written in Phase I carries `CurrencyIsoCode = <CURRENCY>`, AND both buyer `User` rows plus the **Site Guest User** have `DefaultCurrencyIsoCode = <CURRENCY>`. New users inherit the corporate currency at creation — if `<CURRENCY>` ≠ corporate currency you must explicitly set it via Apex (F.3) / `sf data update record` (F.2). Skipping this is the #1 silent cause of storefront `Price Unavailable` when every other pricing piece looks correct.
- [ ] **Phase Z self-validation passed**: the agent ran every Z.1–Z.7 check itself and printed the pass/fail table with **PASS** (or justified **SKIP**) on every row before handing the storefront back to the user.

### Base bootstrap is "done" when Phases A → H **plus Phase Z** pass. Phase I is optional and runs only when the user explicitly asks.

## Commands cheat sheet

| Goal | Command / object |
|------|------------------|
| List authenticated orgs (Phase 0 Q1) | `sf org list --json` |
| Authenticate a new org (Phase 0 Q1) | `sf org login web --alias <new-alias>` |
| Templates | `sf community list template` |
| Create site | `sf community create` |
| Poll async | SOQL **`BackgroundOperation`** |
| Publish | `sf community publish` |
| Clone WebStore scalars | `sf data update record -s WebStore` |
| Payments + tax engine wiring | **`StoreIntegratedService`** (`StoreId`, `Integration`, `ServiceProviderType`) |
| Shipping profiles | **`ShippingConfigurationSet`** (`TargetRecordId` = **`WebStore.Id`**) |
| Product ↔ shipping profile | **`ShippingConfigSetProduct`** (`ShippingProfileId`, `Product2Id`) |
| Search index | `sf commerce search start` |
| Catalog content (Phase I, optional) | `ProductCategory`, `Product2`, `PricebookEntry`, `ProductCategoryProduct`, `CommerceEntitlementProduct`, `ProductMedia` |
