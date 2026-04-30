# What you must do in Salesforce (cannot rely on CLI alone)

CLI + Apex can cover **Networks**, **WebStores**, **catalog & pricebooks**, **buyer groups**, **buyer accounts/contacts**, **buyer Users + permission set assignments** (via anonymous Apex — see `SKILL.md` Phase F), **Site Guest User PSAs**, **metadata deploy** (Experience bundle + navigation), **`sf community publish`**, **`sf commerce search start`**, and any **SOQL-driven setup**.

**Hard rule for the agent:** anything that the CLI or anonymous Apex *can* do automatically **must** be done as part of the bootstrap — the agent never asks the user to assign permission sets, create buyer Users, set `Network` flags, or run `BuyerGroupPricebook` activations by hand.

Below is **only what truly needs human click-through** (Salesforce API genuinely blocks the operation, license/contract limits get in the way, or browser-side validation is required).

---

## 1. Guest buyer profile on the store (API blocked — you click in Commerce)

Salesforce stores this on **`WebStore.GuestBuyerProfileId`**. Admins often **cannot** update it with Data API. You align it in the **Commerce app** (not on the Experience site record).

### Path A — Refreshed Commerce app (Summer ’24+, what Salesforce documents)

This is the chain people miss: you must **pick the store in the sidebar first**.

1. App Launcher → open **Commerce**.
2. In the **left navigation**, use the **Store** dropdown / store picker and select **Cursor Commerce Demo**.  
   - If you do not see a store list, open **Commerce** **Home** and look for **Your stores** / **Stores** and select the store from there, then return to settings.
3. Go to **Settings** → **Store** → **Buyer Access**.  
   **Same destination, different labels:** **Store Settings** → open the **Buyer Access** tab.
4. On **Buyer Access**, find **Guest Access** (guest browsing / guest checkout). Open or expand it.  
   - Salesforce may **create** a guest buyer profile when you enable guest access the first time; to **swap** to an existing profile (e.g. **B2B Guest Buyer Profile**), look for a lookup or step inside **Guest Access** / **guest checkout** setup, or use Path B.
5. **Save** anything you change.

Official topic names in English help: **Manage Guest Access**, **Manage Buyer Access to B2B Stores** (`help.salesforce.com`).

### Path B — Jump straight to Guest Buyer Profiles (compare / open related store)

Use this if Path A does not show a profile dropdown:

1. App Launcher → search **Guest Buyer Profiles** → open the object list.
2. Open **B2B Guest Buyer Profile** (reference) and **Cursor Commerce Demo Guest Buyer Profile** (template) and compare **Related** lists (e.g. **Buyer Groups**, **Stores** if exposed).
3. If **Stores** appears on the profile, open the link to **Cursor Commerce Demo** and look for store-level fields there.

### Path D — Cannot switch profiles in **Guest Access** → mirror buyer groups via CLI

If Commerce only shows **Cursor Commerce Demo Guest Buyer Profile** with no way to pick **B2B Guest Buyer Profile** (rename-only), **parity is still possible**: copy **`BuyerGroupMember`** rows from **`B2B Guest Buyer Profile`** onto the template **`GuestBuyerProfile`** — **`BuyerGroupMember.BuyerId`** accepts **`GuestBuyerProfile`** (same Related list as in the Commerce UI).

Reference profile Id (example org): **`B2B Guest Buyer Profile`**. Resolve yours with SOQL. Then:

```text
sf data query -q "SELECT BuyerGroupId, BuyerGroup.Name FROM BuyerGroupMember WHERE BuyerId = '<B2B_GUEST_BUYER_PROFILE_ID>'" -o <alias>

sf data create record -s BuyerGroupMember -o <alias> -v "BuyerGroupId=<GROUP_ID_FROM_QUERY> BuyerId=<TEMPLATE_GUEST_BUYER_PROFILE_ID>"
```

**Done for Cursor Commerce Demo** (`cursorTesting`): **Cirrus Buyer Group** linked to **`Cursor Commerce Demo Guest Buyer Profile`**.

### Path C — Legacy **Commerce Storefront Console**

If your org still uses the old console (not the refreshed Commerce app), menus differ. In Setup, search **Commerce Storefront Console** or **Enable the Refreshed Commerce App** and switch to the refreshed app so Path A applies.

---

**Goal:** **`WebStore.GuestBuyerProfileId`** for **Cursor Commerce Demo** should match **SDO - B2B Commerce Enhanced** (**`B2B Guest Buyer Profile`**). Verify with SOQL when needed:

```text
sf data query -q "SELECT Name, GuestBuyerProfile.Name FROM WebStore WHERE Name IN ('Cursor Commerce Demo','SDO - B2B Commerce Enhanced')" -o <alias>
```

---

## 2. Experience Cloud site — Members (Commerce permission sets)

**Automated by the agent** as part of `SKILL.md` **Phase F.1**. The agent inserts the `NetworkMemberGroup` rows for the buyer profile and the three Commerce permission sets via `sf data create record`. You do **not** click in **Digital Experiences → Workspaces → Administration → Members** unless the agent reports a specific failure.

UI fallback (only if CLI is blocked in your org): **Digital Experiences** → site → **Workspaces** → **Administration** → **Members** → move **`B2B Commerce - Guest Browser Access`**, **`B2B Commerce - B2B Community Access`**, **`D2C Commerce - Guest User Access`** and the buyer profile to Selected → **Save**.

---

## 3. Wait for commerce search after big catalog/store changes

After catalog, entitlement, or store changes, search must finish indexing.

**Agent can run:** `sf commerce search start --store-name 'Cursor Commerce Demo' --targetusername <alias>`

**You must:** wait until indexing **finishes** (Commerce workspace **Search** / status UI, or org notification). Then test the storefront again as guest.

---

## 4. Test “Login as” correctly (browser)

**You must:**

1. **Setup** → **Users** → open the community user → **Login** → **Login as User**.
2. In the **same browser**, paste your **Experience site** URL (`https://…my.site.com/…/s/…`). Do **not** stay only inside internal Lightning (`*.lightning.force.com`) if you want to see the storefront as that user.

---

## 5. Licenses and permission sets (only when blocked)

If user creation or site access errors mention **license**, **permission**, or **profile** restrictions:

**You must** assign the right **Community / Commerce** licenses and permission sets in **Setup** or **Digital Experiences**. The agent cannot guess your contract limits.

---

## 6. Optional — look & feel only

Changing themes, heroes, images, or rearranging Builder components is **optional** for making checkout work.

**You use:** **Experience Builder** → edit pages → **Publish**.

---

## 7. Publish CMS workspace content to your site channels (UI — Connect REST `publish` returns generic error)

The **Cirrus product images** live in the **`All Commerce - Enhanced`** CMS workspace. The agent already linked your two Cursor channels (Community + PublicUnauthenticated) to that workspace via Connect REST (**`PATCH /connect/cms/spaces/{spaceId}/channels`** with the **`spaceChannels`/`operation: Add`** payload — see SKILL).

Salesforce’s `POST /connect/cms/contents/publish` returns a generic **`Your content wasn’t published. Try again.`** error against this workspace, regardless of single/bulk/`variantIds`/`contextContentSpaceId`. **You must publish from the UI:**

1. App Launcher → **Digital Experiences** → **CMS Workspaces** → **All Commerce - Enhanced**.
2. (Optional, only if any item still says `Draft`) select content (use **Select all** at the top), then **Publish**.
3. **Channels** tab → confirm both **Cursor Commerce Demo** and **Cursor Commerce Demo Channel** are **Added**. They will already appear because the agent linked them.
4. From the **Content** tab, select all content (or only product images), then **Publish** → confirm.
5. Wait for the publish job to finish. Reload the storefront — product images should now render.

If the workspace content is still authored only in another workspace (e.g. **`B2B Commerce Content`**), repeat steps 1–4 against that workspace.

---

## 8. Activate pricing + Priority (automated — verify only)

`SKILL.md` **Phase B** + **Phase E** create three `WebStorePricebook` rows (Standard / Sale / VIP) with `IsActive=true` and four `BuyerGroupPricebook` rows with explicit `Priority=1` (offer) / `Priority=2` (Standard, strikethrough). The agent runs all of this for every new store; without `Priority`, multiple active pricebooks per buyer group cause **`Price Unavailable`** in the storefront.

Verify with:

```text
sf data query -q "SELECT Pricebook2.Name, IsActive FROM WebStorePricebook WHERE WebStoreId = '<WEBSTORE_ID>'" -o <alias>
sf data query -q "SELECT BuyerGroup.Name, Pricebook2.Name, IsActive, Priority FROM BuyerGroupPricebook WHERE BuyerGroupId IN (<STORE_BUYER_GROUP_IDS>) ORDER BY BuyerGroup.Name, Priority" -o <alias>
```

If anything is missing or wrong, ask the agent to re-run Phase E — do **not** patch by hand.

---

## 9. Guest user + buyer user permission sets (automated — verify only)

`SKILL.md` **Phase F.2** assigns the Site Guest User's permission sets (`B2B_Commerce_Guest_Browser_Access`, `D2C_Commerce_Guest_User_Access`) and **Phase F.3** runs an anonymous Apex script that:

1. Inserts the two buyer `User` rows for the Standard + VIP contacts (Apex `insert User` works on sandboxes/DX orgs — `sf org create user` only works on scratch).
2. Assigns **`B2BBuyer`**, **`B2BCommerce_Community_Access`** and **`B2B_Commerce_Customer_Community_Plus_Access`** to each buyer User.
3. Sets a deterministic temporary password so the buyer can log in immediately.

Then **Phase F.4** sets **`Network.OptionsAllowInternalUserLogin=true`**, **`OptionsSelfRegistrationEnabled=true`**, **`OptionsEmbeddedLoginEnabled=true`** and **`SelfRegProfileId`**.

Verify everything in one shot:

```text
sf data query -q "SELECT PermissionSet.Name FROM PermissionSetAssignment WHERE AssigneeId IN (SELECT Id FROM User WHERE Profile.Name = '<Site Name> Profile')" -o <alias>
sf data query -q "SELECT Username, IsActive, ContactId FROM User WHERE ContactId IN ('<STD_CONTACT_ID>','<VIP_CONTACT_ID>')" -o <alias>
sf data query -q "SELECT Assignee.Username, PermissionSet.Name FROM PermissionSetAssignment WHERE AssigneeId IN (SELECT Id FROM User WHERE ContactId IN ('<STD_CONTACT_ID>','<VIP_CONTACT_ID>')) ORDER BY Assignee.Username, PermissionSet.Name" -o <alias>
```

Expected: 2 active buyer Users, 6 buyer-side `PermissionSetAssignment` rows (3 PS × 2 users), guest user has at least the two guest PS.

After publish, **fully sign out and reopen the Experience URL in a new private window** before testing.

> **Self-registration handler:** if the org already has a `CommunitiesSelfRegConfig` Apex class or a custom registration Flow, the agent patches it during Phase F to include the same PS block so every new self-registered buyer is provisioned automatically. If no handler exists, the agent flags it explicitly (this is the only Phase F item that may stay as TODO).

---

## 11. Pending / parked items (do not lose track)

The agent has handed these back to you because either the diagnostic loop is closed without a clean fix from CLI, or the user explicitly deferred them:

### 11.1 Storefront price `Price Unavailable` for logged-in buyers — RESOLVED (2026-04-23)

**Root cause:** the **Salesforce Pricing engine** (`Setup → Salesforce Pricing Setup`) had no Recipe / Procedure assigned for the org. Without it, the buyer-side pricing API returns `PRICE_NOT_FOUND` for every product even when `WebStorePricebook`, `BuyerGroupPricebook`, `PricebookEntry`, `CommerceEntitlementBuyerGroup` and `BuyerAccount` are all valid.

**Fix (UI — the agent does not have a confirmed CLI path yet):**

1. **Setup → Salesforce Pricing Setup**.
2. **Pricing Recipe** → `CommerceDefaultRecipe`.
3. **Pricing Procedure** → `Cirrus - Commerce Default Pricing Procedure`.
4. Click **Sync** and wait for the success toast.
5. Re-run `sf commerce search start --store-name '<Site Name>' --targetusername <alias>` and `sf community publish`.
6. Hard-refresh the storefront in incognito as a buyer — prices should now render.

This step is now part of the standard bootstrap as **`SKILL.md` Phase G.1**. The agent attempts CLI discovery (`PricingProcedure`, `PricingRecipe`, `CommerceConfigRelatedRecord`) before falling back to the UI; if those queries return rows in your org, capture the IDs and the equivalent `sf data create record` commands so the next bootstrap can skip the UI entirely.

**Verification call:**

```text
sf api request rest "/services/data/v62.0/commerce/webstores/<WEBSTORE_ID>/pricing/products/<SAMPLE_PRODUCT_ID>" --target-org <alias>
```

A `200` with `unitPrice` populated confirms the engine is wired. `PRICE_NOT_FOUND` means the Recipe/Procedure are not set or **Sync** wasn't clicked.

### 11.2 Storefront site logo not visible after `StaticResource` swap (parked)

Status: **`SiteLogo` and `_SiteLogoUrl` = `/resource/AscendumLogo`** in `BrandingSet B2B_Commerce`, the resource is deployed and recolored to brand blue **`#003846`** (verified — only `fill="#003846"` and `fill="none"` strokes remain). Bundle deployed and site published, but the user still does not see the logo on the storefront header.

**Read the source first.** The `dxp_content_layout:siteLogo` LWC source is in the [forcedotcom/b2b-commerce-open-source-components](https://github.com/forcedotcom/b2b-commerce-open-source-components/tree/main/force-app/main/default/sfdc_cms__lwc) repo (look for a `siteLogo` / `logo*` folder). Inspecting its `.html` template + `.js` will tell you definitively which `SiteLogo` URL formats it accepts (Static Resource path vs CMS asset path vs external URL) and where the shadow-DOM boundary sits. Forking it under `force-app/main/default/lwc/cursorSiteLogo/` and replacing the reference in `sfdc_cms__view/home/content.json` is the cleanest fix once we pick this back up.

Hypotheses to try when picking this back up:

- **LWR / browser cache.** The `dxp_content_layout:siteLogo` LWC caches the resolved `SiteLogo` URL per build version. Republish, bump the bundle version, and test in a fresh incognito. If the new SVG is being requested at all, DevTools → Network → filter `AscendumLogo` will show a `200`.
- **Resource not public to the guest profile.** `/resource/<Name>` only renders if the **Site Guest User profile** has access to the static resource. Manually verify under **Setup → Profiles → <Guest profile> → Enabled Static Resources**, or grant via permission set. (For authenticated users the **B2B Commerce buyer permission sets** usually cover it.)
- **Wrong path format expected by `dxp_content_layout:siteLogo`.** Some LWR builds expect `SiteLogo` to point to a CMS-managed image (`/sfsites/c/cms/delivery/...`) rather than `/resource/...`. Test by uploading the SVG as a **ContentAsset / `ManagedContentVersion`** in the **CMS workspace linked to the site channels**, then set `SiteLogo` to that asset URL.
- **The SVG renders 0×0.** The recolor script kept `fill="none"` on the root `<svg>`. If the LWC strips dimensions and relies on intrinsic size, force `width="200" height="44"` and a `viewBox` on the root `<svg>` element.

DevTools quick check — paste in the storefront console as the buyer:

```text
$$('img').filter(i => /logo|brand|wordmark/i.test(i.alt + i.src)).map(i => ({src: i.src, w: i.naturalWidth, h: i.naturalHeight}))
```

If `naturalWidth=0`, the asset URL is wrong (404 or blocked). If `naturalWidth>0` but you can’t see it, it’s a contrast / size / `display:none` problem in CSS.

### 11.3 Self-registration handler not yet automated (open)

Newly self-registered buyers still need the three permission sets manually (`B2BBuyer`, `B2BCommerce_Community_Access`, `B2B_Commerce_Customer_Community_Plus_Access`). Add the assignment block inside the **`CommunitiesSelfRegConfig`** Apex class (or whichever self-registration handler / Flow the site is using) so this happens automatically on every new signup.

### 11.4 Product images for the new Ascendum catalog (parked)

The full **`Ascendum Equipment Catalog`** (5 categories, 14 products), **`Ascendum Price Book`** (Standard + Ascendum entries) and **`Ascendum Entitlement Policy`** are wired to **`Cursor Commerce Demo`** — the Cirrus catalog/pricebook/policy have been swapped out cleanly. Storefront will render product cards with **placeholder images** until the CMS image workflow runs.

**Why placeholders today:** B2B Commerce LWR reads product images from `ProductMedia → ManagedContent` (CMS image content type **`sfdc_cms__image`**). External URLs in `Product2.DisplayUrl` are *not* picked up by the product card. The agent tried both the **`POST /connect/cms/contents`** REST endpoint and **`ConnectApi.CmsContent`** in Apex; the org rejects every multipart shape on the REST side ("A request body is required" / "INVALID_API_INPUT") and the Apex types **`ConnectApi.ManagedContentNodeValueInput`** / **`ConnectApi.CmsContent`** are not exposed for compilation in this org's Apex namespace. This is why image upload is parked here instead of fully scripted.

**Manual path that always works (CMS Workspace UI → ProductMedia DML):**

1. **Setup → Digital Experiences → All Sites** → open **All Commerce - Enhanced** workspace (CMS Space `0Zug8000000evT3CAI`).
2. **+ Add Content → Image**, drag the image (or paste URL — Salesforce downloads it). Title it with the product SKU, e.g. **`VOL-L150H Image 1`**.
3. Repeat for the 14 SKUs (image suggestions sourced from `ascendum.pt` are documented in **`SKILL.md` Phase G — Catalog & products**).
4. **Publish** all the new images so the storefront can read them (CMS workspace → Publish, same flow you used for the existing Cirrus images).
5. Get each new `ManagedContent.Id` (starts with `20Y…`) via SOQL:

   ```text
   sf data query -o cursorTesting -q "SELECT Id, Name FROM ManagedContent WHERE Name LIKE 'VOL-%' OR Name LIKE 'VAL-%' OR Name LIKE 'PON-%' OR Name LIKE 'REN-%' OR Name LIKE 'FEN-%' OR Name LIKE 'MF-%' OR Name LIKE 'PRT-%' OR Name LIKE 'SVC-%' ORDER BY Name"
   ```

6. For every product, create a `ProductMedia` row linking `Product2 → ManagedContent`:

   ```text
   sf data create record -s ProductMedia -v "ProductId=<01tg...> ElectronicMediaId=<20Y...> SortOrder=0" -o cursorTesting
   ```

   The map of `Product2.Id` per SKU is in **`/Users/dsiguenza/Documents/SFCore/cursorTesting/catalog-work/product-ids.json`**.

7. Re-run **`sf commerce search start --store-name 'Cursor Commerce Demo' --targetusername cursorTesting`** and **`sf community publish --name 'Cursor Commerce Demo' --target-org cursorTesting`**.

**If you want to re-attempt full automation** (when next picking it up): the `POST /connect/cms/contents` schema for `sfdc_cms__image` requires `contentBody.sfdc_cms:media.source.{type, ref}` where `ref` must be the part name in a multipart upload — the agent verified the JSON shape is correct but the multipart envelope is being rejected. Likely needs Workbench REST Explorer or a small Node.js script using `form-data` package; the same call from Apex would need a SF org with the modern `ConnectApi.CmsContent` namespace exposed (try a developer scratch org from a recent Spring/Summer release).

**Alternative — fork `productCard` to read external URLs.** If the multipart CMS upload keeps blocking automation, the cleanest workaround is to **fork the stock `productCard` LWC** from [forcedotcom/b2b-commerce-open-source-components](https://github.com/forcedotcom/b2b-commerce-open-source-components/tree/main/force-app/main/default/sfdc_cms__lwc) (look for `productCard*` / `searchProductCard*`) so it falls back to **`Product2.DisplayUrl`** (or any custom field with the external URL) when there is no `ProductMedia.ElectronicMediaId`. Then we just set `Product2.DisplayUrl = 'https://ascendum.pt/...'` for each SKU and the storefront renders the image directly through the CSP-trusted domain — no CMS workspace round-trip. Same fork pattern documented in `SKILL.md → Reference resources`.

---

## Quick recap

### Agent / CLI does this automatically (mandatory on every new site)

| Task | Where in `SKILL.md` |
|------|---------------------|
| Create Experience site + WebStore, set `Network` flags (public access, `OptionsEmbeddedLoginEnabled`, `OptionsAllowInternalUserLogin`, `SelfRegProfileId`), publish | Phase A + Phase F.4 |
| Provision the default **3 pricebooks** (Standard / Sale / VIP) on the new store via **`WebStorePricebook`** + `WebStore.StrikethroughPricebookId` | Phase B |
| Provision the default **2 buyer groups + 2 accounts + 2 contacts** (Standard + VIP) and wire them to Sale/VIP pricebooks with `Priority=1` (offer) and `Priority=2` (Standard, strikethrough), plus `CommerceEntitlementBuyerGroup` for both | Phase E |
| `NetworkMemberGroup` rows for the buyer profile + Commerce permission sets | Phase F.1 |
| Site Guest User → assign **`B2B_Commerce_Guest_Browser_Access`** + **`D2C_Commerce_Guest_User_Access`** | Phase F.2 |
| Buyer Users (Standard + VIP) created via anonymous Apex; assign **`B2BBuyer`** + **`B2BCommerce_Community_Access`** + **`B2B_Commerce_Customer_Community_Plus_Access`** to each; set temporary password | Phase F.3 |
| Patch existing self-registration handler (Apex class or Flow) with the same buyer PSAs when present | Phase F.3 (only TODO if no handler exists) |
| Tax / Payments / Shipping / Pricing wiring (`StoreIntegratedService`, `ShippingConfigurationSet`, `ShippingConfigSetProduct`) | Phase C + Phase D |
| Link CMS workspace → site channels (**`PATCH /connect/cms/spaces/{id}/channels`**) | Phase C / Troubleshooting |
| Run `sf commerce search start` + `sf community publish` after every change | Phase H |

### You (UI) — only when the API genuinely blocks it

| Task | Why it's manual |
|------|-----------------|
| **Salesforce Pricing Setup** (Phase G.1) — set Pricing Recipe `CommerceDefaultRecipe` + Pricing Procedure `Cirrus - Commerce Default Pricing Procedure` and click **Sync** | The agent attempts CLI discovery first (`PricingProcedure` / `CommerceConfigRelatedRecord`); fall back to UI when those queries return empty in your org. **This is the most common cause of `Price Unavailable` on the storefront.** |
| Guest buyer profile = **B2B Guest Buyer Profile** on the store (Commerce app) | `WebStore.GuestBuyerProfileId` rejects API writes for most admins |
| **Publish CMS workspace content to site channels** | `POST /connect/cms/contents/publish` returns generic error — UI publish works |
| Confirm search index completed after `commerce search start` | Async indexing, UI status only |
| Login-as → open **Experience URL** in the same browser session | Browser-side test, cannot be scripted |
| Licenses when Setup blocks API | Contract / org-license limits |

### Pending / deferred (next session)

| Task | See |
|------|-----|
| **Site logo not visible on storefront** (parked) | §11.2 |
| **Upload product images for Ascendum catalog and link `ProductMedia`** | §11.4 |

### Recently resolved

| Task | Fix |
|------|-----|
| **`Price Unavailable` for logged-in buyers** (was §11.1) | Salesforce Pricing Setup → Recipe `CommerceDefaultRecipe` + Procedure `Cirrus - Commerce Default Pricing Procedure` + **Sync**. Now baked into Phase G.1. |
